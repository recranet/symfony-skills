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

## Priorities

Every rule in this reference serves these, in order — when they conflict,
the higher one wins:

1. **Atomic failure semantics.** A run either commits whole — bulk writes
   *and* their side effects — or rolls back whole and retries as a unit.
   Never a committed write divorced from a failed side effect that a human
   must notice and reconcile.
2. **Flat memory.** Usage independent of row count. Getting this wrong means
   an OOM crash — or worse, silent no-op flushes on detached entities.
3. **One owner per behavior.** Each side effect lives in exactly one place,
   on the ORM path, regardless of which write path triggered it — duplicated
   logic drifts.

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

## Delivering the skipped side effects: messages, never setters

When a bulk write *does* have side effects you can't drop (status recompute,
detaching a relation, audit logs written by an onFlush subscriber), don't
smuggle logic back into setters or inline it after the bulk statement. Apply
them behind a Messenger domain message (`XxxArchived($ids)`, batched ids):
a handler re-loads each row **fresh** and mutates it through the ORM —
that's exactly what makes the listeners/subscribers fire, so the side-effect
code lives in one place and runs identically regardless of which path
triggered it. The message is the organizational chokepoint — one owner per
side effect — whether or not a queue is involved.

Two facts first, because they anchor everything below: **bulk DQL never
needs a flush** — it executes against the database immediately, bypassing
the UnitOfWork; `flush()` exists only to write changes made through managed
entities. And **handlers flush themselves** — Messenger never flushes for
you; an ORM-mutating handler ends with its own single `$this->em->flush()`.

**Route these messages explicitly to the `sync` transport (`sync://`).**
This is prescribed, not a choice: import side effects are small domain
logic — pure-DB writes in the same database, no external I/O — and sync
dispatch executes the handler in-process at the dispatch site, inside the
import transaction. That satisfies priority 1 by construction, and every
other axis falls in line:

- **One atomic commit.** Bulk write and side effects commit together; a
  handler failure rolls the whole run back, and the run retries as a unit.
  No queue, no gap, no outbox machinery needed.
- **Invariants hold at every observable moment.** No window where a row is
  archived but its dependent state isn't yet — readers (UIs, later import
  phases, other consumers) never have to tolerate half-state.
- **No correctness dependency on infrastructure.** Nothing depends on a
  consumer cron running, a failed queue being monitored, or retries
  eventually succeeding — only on the transaction you're already in.
- **No concurrency.** In a command chaining several importer phases (each
  its own transaction), the handler completes inside phase N before phase
  N+1 starts. Async breaks this: a message committed with phase N becomes
  consumable immediately, so its handler can run *concurrently with phase
  N+1's still-open transaction* — same-table writes from two connections,
  lock waits at best, deadlock-retry noise at worst, an ordering you can't
  reason about.

The one condition sync imposes: the handler runs inside the **dispatcher's
EntityManager**, not a fresh one. It inherits the identity map (its
`find()`s are served stale pre-loaded instances first), its `flush()` writes
the *whole* UnitOfWork (silently committing any half-finished changes the
dispatching code had pending), and its exceptions propagate into the
dispatcher. So **dispatch sync-routed messages only from points where the
EntityManager is clean** — empty or fully flushed, e.g. after the loop's
final flush+clear, as the last step before commit. A dirty dispatch context
is a bug in the dispatch point; fix it there.

**The escape hatch — async on the doctrine transport = the transactional
outbox.** The day a side effect grows external I/O (an API call, mail) —
anything that must not extend the import transaction or couple the import's
success to a third party's uptime — move *that one message* to `async` on
`doctrine://default` over the **same connection** the import writes through,
still dispatched **inside** the open transaction. The dispatch is then just
an INSERT into `messenger_messages` participating in your transaction:
commit → write and message both durable; rollback → neither exists; crash
after commit → the message survives and is consumed. Still no divergence —
but you trade atomicity for isolation: the handler runs at the consumer's
next poll (the row is invisible until your commit — a locking SELECT can't
see uncommitted rows — then 0–60 s with a per-minute `messenger:consume`
cron, unbounded if the worker is down), side effects become *eventually
consistent*, and their failures retry independently via the failed transport
instead of rolling the write back — a failure parked unnoticed in the failed
queue is divergence until a human acts. Accept those costs only for the
message that needs them.

Rules that keep this sound:

- **Never leave a message unrouted.** Symfony handles unrouted messages
  synchronously at the dispatch site too — but *implicitly*. Sync execution
  must be a visible, deliberate routing decision in `messenger.yaml`, not a
  fallthrough; treat a missing routing entry as a bug.
- **Never move an in-transaction dispatch to AMQP/Redis.** A foreign broker
  cannot join the DB transaction — the outbox property silently disappears.
  You'd need a real outbox table + relay first. Leave a warning comment on
  the routing block.
- **Keep handlers on the ORM path — never "narrow" a handler's flush by
  rewriting it in bulk DQL.** The handler exists precisely to *be* the ORM
  path: setter logic, cascades, and the flush that makes onFlush subscribers
  write their logs. DQL there means hand-writing those effects in SQL,
  duplicating subscriber logic that then drifts — the original disease, one
  layer down. DQL for the dumb bulk marker; ORM for behavior. (If a
  dirty-context dispatch worries you, fix the dispatch point, not the
  handler.)
- **Handlers must be idempotent and race-aware**: the doctrine transport
  redelivers on retry, a batched message that fails mid-loop is retried
  whole, and (async) a row can be deleted or restored between dispatch and
  consumption — skip rows that no longer match the precondition.
- Anything that must be atomic with the bulk write belongs in the bulk
  statement itself or a sync-routed handler — never in an async one.
- In tests, make the async transport `in-memory://`: assert dispatches via
  the transport's `getSent()`, and invoke handlers directly to test side
  effects.

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
- Messenger (doctrine transport, routing): https://symfony.com/doc/7.4/messenger.html — see also [messenger.md](messenger.md)
- Console component: https://symfony.com/doc/7.4/console.html
