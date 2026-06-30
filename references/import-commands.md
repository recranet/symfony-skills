# Import / batch-processing commands

## When to use

You're writing a console command that walks a large or growing dataset —
importing from an external API (HubSpot, Sendgrid, a CSV/feed), backfilling
a column, re-syncing, or any "loop over every row" job.

The defining risk is **memory**. A command that loads everything and flushes
once works on today's data and silently dies when the dataset grows:

```
PHP Fatal error: Allowed memory size of 268435456 bytes exhausted
  (tried to allocate 24576 bytes) in .../monolog/src/Monolog/Utils.php
```

The crash often surfaces in an unrelated file (Monolog, the serializer)
because that's just where the *next* allocation happened to land — the real
cause is unbounded growth in the loop. Don't fix it by raising
`memory_limit`; that only moves the cliff. Build the command so memory stays
flat regardless of row count.

## What you need

```
composer require maker --dev  # for make:command
```

Console + Doctrine are already in `symfony/framework-bundle`. See
[console-commands.md](console-commands.md) for command structure and
[doctrine.md](doctrine.md) for the underlying bulk-operation primitives.

## The core problem

Two things grow without bound inside a naive loop:

1. **Doctrine's Unit of Work.** Every entity you `persist()`, `find()`, or
   hydrate via a query stays *managed* until you `clear()`. `flush()` alone
   does not detach them. Loop over 200k rows with one flush at the end and
   you hold 200k managed entities plus their change-set snapshots.
2. **Buffered log records.** Monolog's `fingers_crossed` handler (the prod
   default) buffers every log record until something trips it, and Doctrine's
   SQL logger records every query. A long loop accumulates both.

The fix for (1) is to **flush in batches and `clear()`**, processing by entity
ID so you never hold the whole result set. The fix for (2) is to **disable
SQL logging** for the duration and avoid per-row `info()` logging.

## Minimal example — defensive read loop (ID-first)

This is the pattern from `SendgridVerifyCommand`: select only IDs up front,
then load and process one entity at a time, clearing the EntityManager after
each. Memory stays flat no matter how many rows match.

```php
namespace App\Command;

use App\Repository\ContactRepository;
use App\Service\HubspotImporter;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(
    name: 'hubspot:contacts:import',
    description: 'Sync contacts from HubSpot',
)]
final class HubspotImportCommand extends Command
{
    public function __construct(
        private readonly EntityManagerInterface $em,
        private readonly ContactRepository $contactRepository,
        private readonly HubspotImporter $importer,
    ) {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        // 1. Fetch IDs only — a single lightweight column, not full entities.
        $ids = $this->contactRepository->createQueryBuilder('t0')
            ->select('t0.id')
            ->andWhere('t0.archived IS NULL')
            ->getQuery()
            ->getSingleColumnResult();

        $io->progressStart(\count($ids));

        foreach ($ids as $id) {
            // 2. Load one entity at a time.
            $contact = $this->contactRepository->find($id);
            if ($contact === null) {
                $io->progressAdvance();
                continue;
            }

            // 3. Isolate failures — one bad row must not abort the run.
            try {
                $this->importer->sync($contact);
            } catch (\Throwable $e) {
                $io->warning(sprintf('Contact %s failed: %s', $id, $e->getMessage()));
            }

            // 4. Detach everything loaded this iteration — keeps memory flat.
            $this->em->clear();
            $io->progressAdvance();
        }

        $io->progressFinish();
        $io->success(sprintf('Processed %d contacts.', \count($ids)));

        return Command::SUCCESS;
    }
}
```

Why ID-first instead of `findAll()`: `getSingleColumnResult()` returns a flat
array of scalars (cheap, even for millions of rows). You then hydrate exactly
one entity per iteration and throw it away. `findAll()` would hold every
entity for the whole run.

## Common patterns

### Batch flushing for inserts / updates

When you're *writing* rather than just reading, clearing after every row is
wasteful. Flush and clear every N rows instead:

```php
$batchSize = 100;
$i = 0;

foreach ($rows as $row) {
    $contact = new Contact($row['email']);
    $this->em->persist($contact);

    if (++$i % $batchSize === 0) {
        $this->em->flush();
        $this->em->clear();   // detach the batch just flushed
    }
}

$this->em->flush();           // flush the final partial batch
$this->em->clear();
```

Make the batch size an option so it can be tuned per environment:

```php
->addOption('batch-size', null, InputOption::VALUE_REQUIRED, 'Rows per flush', 100)
```

Tune for the sweet spot: too small = excess round-trips; too large = the
batch itself bloats memory. 50–500 is typical.

> **`clear()` invalidates references.** Any entity you fetched before the
> `clear()` is now detached. If you need a related entity (e.g. a shared
> parent) inside the loop, re-`find()` it after each clear or hold its ID,
> not the object.

**Never resolve a relation once outside the loop.** This is the most common
way the pattern breaks: you fetch the shared `Organization` before the loop to
"avoid re-querying", but the first `clear()` detaches it. Subsequent
iterations associate every entity with a detached object — Doctrine then
either errors (`A new entity was found through the relationship…`) or silently
re-inserts a duplicate organization.

```php
// WRONG — $organization is detached after the first clear().
$organization = $this->orgRepository->find($orgId);

foreach ($ids as $id) {
    $contact = $this->contactRepository->find($id);
    $contact->setOrganization($organization);   // detached after iteration 1
    $this->em->flush();
    $this->em->clear();
}
```

```php
// RIGHT — hold the ID; re-fetch the relation inside the loop, after clear().
foreach ($ids as $id) {
    $contact = $this->contactRepository->find($id);
    $organization = $this->orgRepository->find($orgId);   // managed this iteration
    $contact->setOrganization($organization);
    $this->em->flush();
    $this->em->clear();
}
```

If re-fetching the same parent every iteration is a measurable cost, use
`$this->em->getReference(Organization::class, $orgId)` instead of `find()` —
it returns a lightweight managed proxy without a query, and it too must be
re-acquired after each `clear()`.

### Streaming reads with `toIterable()`

If you must drive the loop from a query result (rather than IDs), use
`toIterable()` so Doctrine hydrates row-by-row instead of building the whole
result array. Still `clear()` periodically:

```php
$q = $this->contactRepository->createQueryBuilder('c')->getQuery();

$i = 0;
foreach ($q->toIterable() as $contact) {
    $this->importer->sync($contact);

    if (++$i % 100 === 0) {
        $this->em->flush();
        $this->em->clear();
    }
}
$this->em->flush();
```

The ID-first pattern is more robust (it survives `clear()` cleanly and
tolerates rows being added/removed mid-run); reach for `toIterable()` only
when re-querying by ID is impractical.

### Silence the SQL logger and the Monolog buffer

Per-query logging is the second-biggest memory sink in a long loop. Turn it
off at the start of the command:

```php
// Doctrine DBAL: stop accumulating the SQL logger stack (dev/profiler).
$this->em->getConnection()->getConfiguration()->setMiddlewares([]);
```

Also avoid logging inside the loop. A `fingers_crossed` Monolog handler
buffers every record until it's triggered; a per-row `$logger->info()` over
100k rows *is* the leak. Log a summary before and after the loop, not per row.
If the import truly needs per-row audit logs, configure a non-buffering
channel for it.

### Memory guard as a safety net

Defensive batching keeps memory flat, but a guard turns a silent OOM crash
into a clean, debuggable exit — useful while a new import beds in. This is the
belt to batching's braces, **not a replacement** for it:

```php
private function memoryLimitBytes(): int
{
    $limit = \ini_get('memory_limit');
    if ($limit === '-1') {
        return \PHP_INT_MAX;
    }
    $value = (int) $limit;
    return match (strtoupper(substr($limit, -1))) {
        'G' => $value * 1024 ** 3,
        'M' => $value * 1024 ** 2,
        'K' => $value * 1024,
        default => $value,
    };
}
```

```php
$ceiling = (int) ($this->memoryLimitBytes() * 0.9);

foreach ($ids as $id) {
    // … process, flush, clear …

    if (memory_get_usage(true) > $ceiling) {
        $io->warning(sprintf(
            'Memory at %s of limit after %d rows — stopping cleanly. Re-run to continue.',
            $this->formatBytes(memory_get_usage(true)),
            $processed,
        ));
        break;   // or return Command::FAILURE
    }
}
```

If you hit the guard *with* batch flushing in place, something is still being
retained per iteration — a growing array, an event subscriber holding
references, or a service caching results. Find it rather than raising the
ceiling.

> Why this over watching memory in the Symfony profiler: the profiler shows
> you *one run* on *today's* data. Defensive batching + a guard hold for every
> future run as the dataset grows. Monitor to discover a leak; program
> defensively so the leak can't take the process down.

### Resumable / chunked imports

For very large or remote sources, make the command resumable so a failure
doesn't force a full restart:

- Page the source API and persist a cursor (last-seen id / page token).
- Add `--since` / `--limit` options so it can run incrementally or be split.
- Make `sync()` idempotent (upsert by external key) so re-running is safe.

### Dry-run

Imports mutate data; a `--dry-run` lets you validate counts and surface
errors without writing:

```php
->addOption('dry-run', null, InputOption::VALUE_NONE, 'Process without persisting')
```

```php
if (!$dryRun) {
    $this->em->flush();
}
$this->em->clear();
```

### Progress and reporting

Use `SymfonyStyle` progress helpers (`progressStart` / `progressAdvance` /
`progressFinish`, or `progressIterate`) and finish with a summary — counts
processed, skipped, failed. See [console-commands.md](console-commands.md).

## Gotchas

- **`flush()` does not free memory — `clear()` does.** The Unit of Work keeps
  every managed entity until cleared. A loop that only flushes still leaks.
- **`clear()` detaches *everything*.** Re-`find()` shared/parent entities
  after clearing, or work from IDs. A stale reference flushed after a clear
  re-inserts or errors.
- **Raising `memory_limit` is not a fix.** It moves the cliff a few months
  out. Batch + clear is the fix.
- **Per-row logging is a leak.** `fingers_crossed` Monolog and the Doctrine
  SQL logger both buffer. Disable SQL logging for the run; log summaries, not
  rows.
- **Don't hold full entities just to read one field.** Select the scalar
  (`getSingleColumnResult()`, DQL `NEW`, partial hydration) instead.
- **Cascade/listener fan-out.** A Doctrine listener or `cascade: persist` can
  pull large object graphs into the Unit of Work you didn't intend — another
  reason to `clear()` between batches.
- **`exit()` vs exit codes.** Return `Command::SUCCESS` / `FAILURE`, never
  `exit()` — wrappers and cron need the code (see
  [console-commands.md](console-commands.md)).

## Docs

- Doctrine batch processing: https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/batch-processing.html
- Console component: https://symfony.com/doc/7.4/console.html
- Console style guide: https://symfony.com/doc/7.4/console/style.html
- Monolog `fingers_crossed`: https://symfony.com/doc/7.4/logging.html
