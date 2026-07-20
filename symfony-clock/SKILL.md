---
name: symfony-clock
description: Testable time with symfony/clock - ClockInterface, ClockAwareTrait, MockClock in tests. Use whenever Symfony code needs 'now' (new DateTimeImmutable, time()) or a test needs to control or freeze time.
---

# symfony/clock

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

Any time you need "now". Reading the clock through a service makes time-
dependent code testable.

## Replaces

- `new \DateTimeImmutable()` / `new \DateTime()` in business logic
- `time()`, `microtime(true)`
- Custom "clock" wrappers

## Install

```
composer require symfony/clock
```

The `Symfony\Component\Clock\ClockInterface` service is autowired to a
`NativeClock` by default.

## Minimal example

```php
use Symfony\Component\Clock\ClockInterface;

public function __construct(private ClockInterface $clock) {}

public function isExpired(\DateTimeImmutable $deadline): bool
{
    return $this->clock->now() > $deadline;
}
```

## Common patterns

### Global helper (when DI is overkill)

```php
use function Symfony\Component\Clock\now;

$now = now(); // \DateTimeImmutable in the current timezone
```

`now()` reads the same clock as `ClockInterface` — including the mock in
tests.

### Testing with `MockClock`

```php
use Symfony\Component\Clock\MockClock;
use Symfony\Component\Clock\Test\ClockSensitiveTrait;

final class SubscriptionTest extends KernelTestCase
{
    use ClockSensitiveTrait;

    public function testExpiry(): void
    {
        $clock = self::mockTime('2026-01-01 00:00:00');
        // … create subscription valid for 30 days …

        $clock->sleep(86_400 * 31); // jump 31 days
        $this->assertTrue($subscription->isExpired());
    }
}
```

`ClockSensitiveTrait` rewires the container's `ClockInterface` to the mock
for the test, and restores it afterwards.

### Fixed timezone

```php
use Symfony\Component\Clock\NativeClock;

$clock = new NativeClock('Europe/Amsterdam');
$clock->now()->getTimezone()->getName(); // 'Europe/Amsterdam'
```

### `PSR-20` compatibility

`ClockInterface` extends `Psr\Clock\ClockInterface` — code expecting the PSR
type works with the Symfony service.

## Gotchas

- Don't inject `\DateTimeImmutable` directly — inject `ClockInterface` and
  call `now()`. The former is fixed at construction; the latter advances.
- In tests, `MockClock` only takes effect for services that depend on
  `ClockInterface`. Code that calls `new \DateTimeImmutable()` directly is
  unaffected — that's the point.
- `MockClock::sleep()` is virtual (no real sleep). Use it freely.

## Docs

https://symfony.com/doc/7.4/components/clock.html
