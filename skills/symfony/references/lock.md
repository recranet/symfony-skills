# symfony/lock

## When to use

You need to prevent concurrent execution of a code path — either across
processes on one host or across hosts in a cluster. Use cases: cron jobs that
must not overlap, deduplicating webhook handling, serialising access to a
shared resource.

## Replaces

- Hand-rolled Redis `SETNX` / `SET … NX EX` wrappers
- `flock()` on lockfiles
- Vendor-specific lock libraries (e.g., `php-lock/lock`, custom mutex classes)
- Database `SELECT … FOR UPDATE` used purely as a mutex

## Install

```
composer require symfony/lock
```

With Flex this also drops `config/packages/lock.yaml` and a `LOCK_DSN` env var.

## Minimal example

```php
use Symfony\Component\Lock\LockFactory;

public function __construct(private LockFactory $lockFactory) {}

public function run(): void
{
    $lock = $this->lockFactory->createLock('invoice-generation');
    if (!$lock->acquire()) {
        return; // someone else is already generating invoices
    }
    try {
        $this->generateInvoices();
    } finally {
        $lock->release();
    }
}
```

`LockFactory` is autowired by Flex from the default `lock` store
(`%env(LOCK_DSN)%`, defaults to a flock store).

## Common patterns

### Blocking acquire with timeout

```php
$lock = $this->lockFactory->createLock('export');
$lock->acquire(blocking: true); // waits forever
// or, with a deadline:
if (!$lock->acquireBlocking(timeout: 30.0)) {
    throw new \RuntimeException('Could not acquire export lock in 30s');
}
```

### Auto-expiring lock (recommended for crashable work)

```php
// expires after 300s even if the holder dies; refresh inside the work
$lock = $this->lockFactory->createLock('long-job', ttl: 300.0, autoRelease: true);
$lock->acquire();
try {
    foreach ($items as $item) {
        $lock->refresh();
        $this->process($item);
    }
} finally {
    $lock->release();
}
```

### Multiple named stores

Use named lock factories when different lock semantics are needed (e.g., a
Redis-backed cluster lock for cross-host, plus a local flock for per-host
work):

```yaml
# config/packages/lock.yaml
framework:
    lock:
        cluster: '%env(REDIS_DSN)%'
        local: 'flock'
```

Autowire by parameter name:

```php
public function __construct(
    private LockFactory $clusterLockFactory,
    private LockFactory $localLockFactory,
) {}
```

## Gotchas

- Without a TTL, a crashed process leaves a held lock until the store is
  cleared. For anything that could crash, set a TTL and call `refresh()`.
- The default Flex store is `flock` — single-host only. For multi-host, set
  `LOCK_DSN` to a Redis, PDO, or Zookeeper DSN.
- `acquire(true)` is blocking and has no timeout. Use `acquireBlocking()` with
  an explicit timeout for production code.
- Locks are not re-entrant. Acquiring the same lock twice in the same process
  is undefined.

## Docs

https://symfony.com/doc/7.4/lock.html
