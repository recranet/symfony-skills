---
name: symfony-scheduler
description: Recurring work with symfony/scheduler - #[AsPeriodicTask] and #[AsCronTask], schedules, Messenger integration. Use whenever a Symfony task should run on an interval or cron expression, even if the user asks for a crontab entry.
---

# symfony/scheduler

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need to run recurring work — every minute, every hour, every Monday at
03:00 — from inside the app rather than from system cron.

## Replaces

- `crontab` files pointing at `bin/console <cmd>` for app-internal jobs
- Custom "scheduled jobs" tables
- Third-party schedulers (`dragonmantank/cron-expression` wrappers)

## Install

```
composer require symfony/scheduler
```

Scheduler runs on top of Messenger — install that too if not present.

## Minimal example

Define a message:

```php
final class GenerateDailyReport {}
```

Define its handler:

```php
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class GenerateDailyReportHandler
{
    public function __invoke(GenerateDailyReport $msg): void
    {
        // …
    }
}
```

Declare the schedule:

```php
use Symfony\Component\Scheduler\Attribute\AsSchedule;
use Symfony\Component\Scheduler\RecurringMessage;
use Symfony\Component\Scheduler\Schedule;
use Symfony\Component\Scheduler\ScheduleProviderInterface;

#[AsSchedule('default')]
final class AppSchedule implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        return (new Schedule())
            ->add(
                RecurringMessage::cron('0 3 * * *', new GenerateDailyReport()),
                RecurringMessage::every('5 minutes', new HealthCheck()),
            );
    }
}
```

Run it:

```
php bin/console messenger:consume scheduler_default -vv
```

## Common patterns

### Multiple schedules (group jobs)

```php
#[AsSchedule('reports')]
final class ReportsSchedule implements ScheduleProviderInterface { /* ... */ }

#[AsSchedule('housekeeping')]
final class HousekeepingSchedule implements ScheduleProviderInterface { /* ... */ }
```

Consume each independently:

```
php bin/console messenger:consume scheduler_reports
php bin/console messenger:consume scheduler_housekeeping
```

### Lock to prevent overlapping execution

```php
return (new Schedule())
    ->lock($lockFactory->createLock('schedule-default'))
    ->add(/* … */);
```

Pair with `symfony/lock` to ensure only one worker runs each tick (important
when running multiple consumers).

### Catch-up after downtime

```php
RecurringMessage::cron('@daily', new GenerateDailyReport())
    ->withJitter(60); // spread across 0-60s to avoid thundering herd

return (new Schedule())
    ->add(/* … */)
    ->processOnlyLastMissedRun(); // skip backlog after long downtime
```

### Stateful schedule (last-run timestamp)

```php
return (new Schedule())
    ->stateful($cachePool)
    ->add(/* … */);
```

Persists state so a restart doesn't re-fire jobs already executed.

## Gotchas

- The Scheduler is in-process. You still need a long-running worker
  (`messenger:consume scheduler_<name>`). It is not a replacement for
  the operating system's cron if no worker is running.
- Without `->lock()`, running multiple consumers will double-fire jobs.
- `processOnlyLastMissedRun()` is essential if downtime is possible —
  otherwise a long outage queues a flood of catch-up messages.
- Cron expressions use standard 5-field syntax; macros (`@daily`, `@hourly`)
  are supported.

## Docs

https://symfony.com/doc/7.4/scheduler.html
