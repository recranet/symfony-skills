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
