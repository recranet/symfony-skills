# Import / batch-processing commands

## When to use

You're writing a console command that walks a large or growing dataset —
importing from an external API (HubSpot, Sendgrid, a CSV/feed), backfilling
a column, re-syncing, or any "loop over every row" job.

The defining risk is **memory**. Every entity you `persist()`, `find()`, or
hydrate stays managed by Doctrine's Unit of Work until you `clear()` —
`flush()` alone does not detach anything. A command that loads everything and
flushes once works on today's data and dies with `Allowed memory size
exhausted` when the dataset grows (the fatal often surfaces in an unrelated
file like Monolog — that's just where the next allocation landed). Don't raise
`memory_limit`; that only moves the cliff.

## The pattern

Select **IDs only** up front, then load one entity per iteration. Call
`$em->clear()` **before the import** (start from an empty Unit of Work) and
**after each iteration** (detach everything that iteration loaded). Memory
stays flat regardless of row count.

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    $io = new SymfonyStyle($input, $output);

    // Silence the SQL logger — it buffers every query otherwise.
    $this->em->getConnection()->getConfiguration()->setMiddlewares([]);

    // IDs only: a flat scalar array is cheap, even for millions of rows.
    $ids = $this->contactRepository->createQueryBuilder('t0')
        ->select('t0.id')
        ->andWhere('t0.archived IS NULL')
        ->getQuery()
        ->getSingleColumnResult();

    $this->em->clear();
    $io->progressStart(\count($ids));

    foreach ($ids as $id) {
        $contact = $this->contactRepository->find($id);

        // Isolate failures — one bad row must not abort the run.
        try {
            if ($contact !== null) {
                $this->importer->sync($contact);
                $this->em->flush();
            }
        } catch (\Throwable $e) {
            $io->warning(sprintf('Contact %s failed: %s', $id, $e->getMessage()));
        }

        $this->em->clear();
        $io->progressAdvance();
    }

    $io->progressFinish();
    $io->success(sprintf('Processed %d contacts.', \count($ids)));

    return Command::SUCCESS;
}
```

`clear()` every iteration also bounds each `flush()` to that iteration's
changes — otherwise `flush()` recomputes change sets for *every* managed
entity (the classic "import starts fast, then crawls", O(n²)) and a row that
failed mid-iteration can be silently re-flushed by a later one.

## Fetch every relation inside the loop. No exceptions.

`clear()` detaches *everything*, so any entity resolved before it is stale.
Make re-fetching an unconditional habit, not a judgement call — the most
common breakage is resolving a shared parent once before the loop "to avoid
re-querying": after the first `clear()` every iteration associates entities
with a detached object, and Doctrine errors (`A new entity was found through
the relationship…`) or silently inserts a duplicate.

```php
// WRONG — $organization is detached after the first clear().
$organization = $this->orgRepository->find($orgId);
foreach ($ids as $id) { ... }

// RIGHT — hold the ID; re-fetch inside the loop, every iteration.
foreach ($ids as $id) {
    $contact = $this->contactRepository->find($id);
    $organization = $this->orgRepository->find($orgId);
    $contact->setOrganization($organization);
    $this->em->flush();
    $this->em->clear();
}
```

The per-iteration query is cheap. If profiling ever shows it hot, use
`$this->em->getReference(Organization::class, $orgId)` (managed proxy, no
query) — but keep it inside the loop; `clear()` invalidates references too.

## Dropping to DBAL / bulk DQL: check for omitted side effects

If raw write volume is genuinely the bottleneck (the ORM issues one `INSERT`
per `persist()` — batching never merges statements), drop to DBAL for
multi-row `INSERT`, `INSERT ... ON DUPLICATE KEY UPDATE` / `COPY`, or use bulk
DQL `UPDATE`/`DELETE`. But raw SQL and bulk DQL **bypass the ORM entirely** —
before using them, check what the entity relies on and replicate it or accept
its absence deliberately:

- **Setter/getter logic** — normalization, slugs, hashing, derived fields
  computed in setters never run; you write raw column values.
- **Lifecycle callbacks and entity/event listeners** — `prePersist`,
  `preUpdate`, `postPersist`, timestampable/blameable/sluggable behaviors,
  audit listeners: none fire.
- **Cascades and orphan removal** — `cascade: persist/remove` and
  `orphanRemoval` are ORM features; DBAL writes touch only the table named.
- **Already-loaded entities go stale** — a bulk DQL `UPDATE` doesn't update
  managed objects in memory; `clear()` after it.

If any of those must run, stay on the ORM path. Mixing is fine too: DBAL for
the dumb bulk part, ORM for rows that need the side effects.

## Odds and ends

- **Per-row logging is a leak.** `fingers_crossed` Monolog buffers every
  record; log summaries before/after the loop, not per row.
- **`--dry-run`** (`InputOption::VALUE_NONE`): skip `flush()`, still
  `clear()` — validates counts without writing.
- **Resumability** for very large/remote sources: page the source, persist a
  cursor, make `sync()` idempotent (upsert by external key), add
  `--since`/`--limit`.
- **Profile before tuning** — for API imports, network latency and rate
  limits usually dwarf the DB writes. `bin/console --profile -vvv <command>`,
  see [debugging.md](debugging.md).
- **Return `Command::SUCCESS`/`FAILURE`, never `exit()`** — see
  [console-commands.md](console-commands.md).

## Docs

- Doctrine batch processing: https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/batch-processing.html
- DQL bulk UPDATE/DELETE: https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/dql-doctrine-query-language.html#update-queries
- Console component: https://symfony.com/doc/7.4/console.html
