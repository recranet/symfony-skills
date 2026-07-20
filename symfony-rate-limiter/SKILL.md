---
name: symfony-rate-limiter
description: Rate limiting with symfony/rate-limiter - token bucket, fixed and sliding windows, per-user/IP limiters, reservations. Use whenever a Symfony task involves throttling, quotas, login attempt limits, or API rate limits, even if the user proposes counting requests in Redis.
---

# symfony/rate-limiter

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need to limit how often something can happen — login attempts, API
calls per user, expensive endpoints, outbound calls to a flaky third-party.

## Replaces

- Hand-rolled "store count + timestamp in Redis" rate limiters
- Third-party rate-limit packages

## Install

```
composer require symfony/rate-limiter
```

## Minimal example

```yaml
# config/packages/rate_limiter.yaml
framework:
    rate_limiter:
        anonymous_api:
            policy: 'fixed_window'
            limit: 100
            interval: '1 hour'
```

```php
use Symfony\Component\RateLimiter\RateLimiterFactory;

public function __construct(private RateLimiterFactory $anonymousApiLimiter) {}

public function handle(Request $request): Response
{
    $limiter = $this->anonymousApiLimiter->create($request->getClientIp());
    if (false === $limiter->consume()->isAccepted()) {
        return new Response('Too many requests', 429);
    }
    // ... handle the request
}
```

The factory service id is `limiter.<name>`; autowire by parameter name
`<name>Limiter`.

## Common patterns

### Policies

- `fixed_window` — N events per interval, resets at interval boundary. Simple.
- `sliding_window` — N events in any rolling interval. Smoother.
- `token_bucket` — burst-capable, refill at a steady rate.
- `no_limit` — allow always (handy for env-conditional configs).

```yaml
framework:
    rate_limiter:
        api_burst:
            policy: 'token_bucket'
            limit: 60
            rate: { interval: '1 second', amount: 1 }
```

### Consume multiple tokens

```php
$limiter->consume(5); // costs 5 tokens (e.g., expensive operation)
```

### Reserve (wait for slot)

```php
$reservation = $limiter->reserve(tokens: 1, maxTime: 5.0);
$reservation->wait(); // sleeps up to 5s if rate limit would otherwise reject
```

Useful for outbound calls where you want backpressure rather than failure.

### Custom storage (defaults to cache pool)

```yaml
framework:
    rate_limiter:
        login:
            policy: 'fixed_window'
            limit: 5
            interval: '15 minutes'
            cache_pool: 'cache.app'  # or a dedicated pool
```

### Login throttling

Symfony's security layer integrates directly:

```yaml
# config/packages/security.yaml
security:
    firewalls:
        main:
            login_throttling:
                max_attempts: 5
                interval: '15 minutes'
```

This uses a built-in limiter — no manual wiring needed.

## Gotchas

- The factory creates per-key limiters. Pass a stable key (user id, IP,
  email). Without a key, all consumers share one bucket.
- `consume()` is non-blocking. To wait, use `reserve()` + `wait()`.
- Token-bucket limits are about steady rate + burst, not "N per interval".
  Use `fixed_window` or `sliding_window` if you want the latter semantics.
- The cache pool must be shared across processes for the limit to be
  cluster-wide. The default Flex pool is fine.

## Docs

https://symfony.com/doc/7.4/rate_limiter.html
