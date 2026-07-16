# symfony/messenger

## When to use

You need to dispatch work asynchronously — background jobs, fan-out, cross-
service events, retries with backoff. Transports: AMQP, Redis, Doctrine,
Amazon SQS, Beanstalkd, in-memory.

## Replaces

- Direct `php-amqplib` use
- Vendor queue SDKs (SQS, Redis-as-queue) when you'd otherwise wrap them
- Hand-rolled "job table" implementations in Doctrine
- Custom event bus implementations

## Install

```
composer require symfony/messenger
# transport bridges as needed:
composer require symfony/amqp-messenger
composer require symfony/redis-messenger
composer require symfony/doctrine-messenger
composer require symfony/amazon-sqs-messenger
```

## Minimal example

Define a message (plain DTO):

```php
final class SendReportMessage
{
    public function __construct(public readonly int $userId) {}
}
```

Define a handler:

```php
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class SendReportHandler
{
    public function __invoke(SendReportMessage $message): void
    {
        // do the work
    }
}
```

Dispatch:

```php
use Symfony\Component\Messenger\MessageBusInterface;

public function __construct(private MessageBusInterface $bus) {}

$this->bus->dispatch(new SendReportMessage(userId: 42));
```

## Common patterns

### Route messages to an async transport

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async: '%env(MESSENGER_TRANSPORT_DSN)%'
            failed: 'doctrine://default?queue_name=failed'
        failure_transport: failed
        routing:
            App\Message\SendReportMessage: async
```

Run the worker:

```
php bin/console messenger:consume async -vv
```

### Retry with backoff

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 5
                    delay: 1000
                    multiplier: 2
                    max_delay: 60000
```

### Failed-message inspection

```
php bin/console messenger:failed:show
php bin/console messenger:failed:retry
php bin/console messenger:failed:remove <id>
```

### Stamps (metadata / control)

```php
use Symfony\Component\Messenger\Stamp\DelayStamp;
use Symfony\Component\Messenger\Envelope;

$this->bus->dispatch(new Envelope($message, [
    new DelayStamp(5000), // delay 5s
]));
```

### Multiple buses (command/query/event)

```yaml
framework:
    messenger:
        default_bus: command.bus
        buses:
            command.bus: ~
            query.bus: ~
            event.bus:
                default_middleware: allow_no_handlers
```

Inject by name: `MessageBusInterface $queryBus`.

## Conventions — flush ownership, routing, sync vs async

For the *why* behind sync-vs-async dispatch and the transactional outbox,
read [imports.md](imports.md) ("Delivering the skipped side effects"). These
are the operating rules that follow from it.

### Flush ownership follows EntityManager ownership

- A **sync-routed handler** executes at the dispatch site, inside the
  dispatcher's EntityManager and transaction. It borrows that EM, so it
  **never flushes** (and never injects the EM for that purpose): `flush()`
  writes the *whole* UnitOfWork, and only the dispatcher knows what else is
  pending in it. The **dispatch site flushes directly below the dispatch**,
  where the UnitOfWork state is known:

  ```php
  $this->bus->dispatch(new OrderArchived($ids)); // sync handler mutates entities
  $this->em->flush();                            // dispatcher owns the write
  ```

- An **async handler** runs in its own consumer process with its own
  EntityManager context. It owns that context, so it **flushes itself**
  (directly, or via a service that does).

A handler must be written for the routing it has — routing is part of the
handler's contract, hence the migration checklist below.

### Routing rules

- **Every message class is routed explicitly in `messenger.yaml`.** Symfony
  handles unrouted messages synchronously at the dispatch site — same
  behaviour as `sync://`, but invisible in config. Sync must be a visible,
  deliberate decision; treat a missing routing entry as a bug.
- `sync://`: small, local, pure-DB side effects that must commit atomically
  with their trigger.
- Async on `doctrine://`: background work, consumed by `messenger:consume`
  from cron or Supervisor. A failure parked in the failed transport
  unnoticed is divergence — check it when async side effects seem missing.
- **Never move an in-transaction dispatch to AMQP/Redis** — a foreign
  broker cannot join the DB transaction, so the outbox property silently
  disappears; you'd need a real outbox table + relay first.

### Sync dispatch preconditions

A sync handler inherits the dispatcher's identity map and transaction, and
its exceptions propagate into the dispatcher. **Dispatch sync-routed
messages only from points where the EntityManager is clean** (empty or
fully flushed UnitOfWork). A dirty dispatch context is a bug in the dispatch
point — fix it there, never by rewriting the handler in DQL: the handler
exists precisely to be the ORM path (setter logic, cascades, the flush that
fires onFlush subscribers). DQL for the dumb bulk marker; ORM for behavior.

### Moving a message sync → async (the escape hatch)

Do this when a side effect grows external I/O or becomes too slow/fragile
to run inside the dispatcher's transaction. It is a behavioural migration,
not a one-line routing edit:

- [ ] Route the message to async on `doctrine://` over the same connection
      the dispatcher writes through, still dispatched inside the transaction
      → the dispatch degrades gracefully into the transactional outbox
      (message row commits with the state change).
- [ ] **Transfer flush ownership**: the handler now owns its EM context, so
      add its own `flush()` — the dispatch-site flush no longer covers it.
- [ ] Verify idempotency and race guards: the doctrine transport redelivers
      on retry, batched messages retry whole, and rows can be deleted or
      restored between dispatch and consumption — skip rows that no longer
      match the precondition.
- [ ] Accept (and document at the dispatch site) the new semantics: eventual
      consistency, failures retry independently instead of rolling back the
      trigger, possible interleaving with work that runs after the
      dispatching transaction commits.

### Testing conventions

Make async transports `in-memory://` in the test env and assert dispatches
via `messenger.transport.async` → `getSent()`. Sync-routed handlers run
inline — assert the resulting state directly instead of intercepting the
message. To test an async handler's side effects, invoke the handler
directly with a constructed message.

## Gotchas

- Workers must be restarted on deploy (`messenger:stop-workers` or a Supervisor
  signal). Symfony will not pick up new code in long-running workers.
- `#[AsMessageHandler]` on `__invoke` is the modern style — don't implement
  `MessageHandlerInterface` (removed in 6.0+).
- Set `failure_transport` so failed messages don't loop forever.
- The Doctrine transport is fine for low/medium throughput; for high
  throughput use AMQP/SQS/Redis.
- In tests, route everything to the `in-memory://` transport.

## Docs

https://symfony.com/doc/7.4/messenger.html
