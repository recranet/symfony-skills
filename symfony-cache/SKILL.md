---
name: symfony-cache
description: Cache expensive work with symfony/cache - pools, cache contracts, stampede protection, tags, Redis/Memcached/APCu backends. Use for any caching task in a Symfony project, even if the user asks for a direct Redis or Memcached client.
---

# symfony/cache

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need to cache the result of an expensive computation, API call, or query.
Works across processes (Redis, Memcached, filesystem) or in-process (array).

## Replaces

- Direct `Predis\Client` / `Redis` extension usage for caching
- Direct `Memcached` extension usage
- `apcu_fetch`/`apcu_store` calls
- Ad-hoc file-based cache helpers

## Install

```
composer require symfony/cache
```

Flex provides `config/packages/cache.yaml` and wires `CacheInterface`.

## Minimal example

```php
use Symfony\Contracts\Cache\CacheInterface;
use Symfony\Contracts\Cache\ItemInterface;

public function __construct(private CacheInterface $cache) {}

public function getExchangeRate(string $from, string $to): float
{
    return $this->cache->get("rate.$from.$to", function (ItemInterface $item): float {
        $item->expiresAfter(3600);
        return $this->fetchRateFromApi($from, $to);
    });
}
```

The closure runs only on miss. Hits are returned directly.

## Common patterns

### Named pool (separate TTLs / backends per concern)

```yaml
# config/packages/cache.yaml
framework:
    cache:
        pools:
            app.users.cache:
                adapter: cache.adapter.redis
                default_lifetime: 300
            app.reports.cache:
                adapter: cache.adapter.filesystem
                default_lifetime: 86400
```

Autowire by parameter name (Symfony rewrites the service id):

```php
public function __construct(private CacheInterface $usersCache) {}
```

### Tag-based invalidation

```yaml
framework:
    cache:
        pools:
            app.product.cache:
                adapter: cache.adapter.redis
                tags: true
```

```php
use Symfony\Contracts\Cache\TagAwareCacheInterface;

$this->productCache->get('product.42', function (ItemInterface $item) {
    $item->expiresAfter(3600);
    $item->tag(['products', 'product-42']);
    return $this->loadProduct(42);
});

// Invalidate everything tagged 'products':
$this->productCache->invalidateTags(['products']);
```

### Delete / contains

```php
$this->cache->delete('rate.USD.EUR');
```

For lower-level access (PSR-6 / PSR-16), inject
`Psr\Cache\CacheItemPoolInterface` or `Psr\SimpleCache\CacheInterface`.

## Gotchas

- `get()` is "compute on miss, store, return" — the closure is the source of
  truth. Don't manually `set()` unless you need to.
- The `$beta` parameter to `get()` enables probabilistic early expiration to
  avoid thundering herds. Leave at default 1.0 unless you understand it.
- Keys must match `[A-Za-z0-9_.]` and be ≤64 chars; use a hash for arbitrary
  inputs.
- Tag invalidation requires a tag-aware pool. Plain `CacheInterface` won't
  expose `invalidateTags()`.
- For HTTP response caching across requests, look at `HttpCache` /
  `cache.HttpCache` separately — this component is for application data.

## Docs

https://symfony.com/doc/7.4/cache.html
