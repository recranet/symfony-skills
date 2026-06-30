# Events — listeners, subscribers, custom events

## When to use

You want to react to something happening elsewhere — a kernel event, a
Doctrine event, an in-app domain event — without coupling the producer to
the consumer.

## What you need

`symfony/event-dispatcher` ships with `symfony/framework-bundle`. No extra
install.

## Minimal example — listener with attribute

```php
namespace App\EventListener;

use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;

#[AsEventListener(event: ExceptionEvent::class, priority: 0)]
final class JsonErrorListener
{
    public function __invoke(ExceptionEvent $event): void
    {
        $request = $event->getRequest();
        if (!str_starts_with($request->getPathInfo(), '/api/')) {
            return;
        }
        // … shape the response
    }
}
```

The `#[AsEventListener]` attribute + autoconfiguration registers the
service — no YAML tags needed.

## Common patterns

### Multiple events on one class — subscriber

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class ApiSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::REQUEST  => ['onRequest', 10],
            KernelEvents::RESPONSE => 'onResponse',
        ];
    }

    public function onRequest(RequestEvent $event): void { /* … */ }
    public function onResponse(ResponseEvent $event): void { /* … */ }
}
```

Use `AsEventListener` per-method when there are 1–3 events; use
`EventSubscriberInterface` when one class owns many event hooks.

### `make:subscriber`

```
bin/console make:subscriber ApiSubscriber
```

### Kernel events (most common)

| Event | When | Use for |
|-------|------|---------|
| `KernelEvents::REQUEST` | Request received | Authn, routing tweaks, locale |
| `KernelEvents::CONTROLLER` | Controller resolved | Controller decoration |
| `KernelEvents::VIEW` | Controller returned non-Response | Convert array → Response |
| `KernelEvents::RESPONSE` | Response built | Headers, content rewriting |
| `KernelEvents::TERMINATE` | After response sent | Slow side-effects (mail, log) |
| `KernelEvents::EXCEPTION` | Uncaught exception | Error handling, API JSON shapes |
| `KernelEvents::FINISH_REQUEST` | Sub-request done | Restore locale |

For `KernelEvents::TERMINATE` to actually fire, the server must be using
`HttpKernel::terminate()` — FrankenPHP, FPM, and the built-in server all do.

### Doctrine events

Prefer `#[AsDoctrineListener]`:

```php
use Doctrine\Bundle\DoctrineBundle\Attribute\AsDoctrineListener;
use Doctrine\ORM\Event\PrePersistEventArgs;
use Doctrine\ORM\Events;

#[AsDoctrineListener(event: Events::prePersist)]
final class StampCreatedAt
{
    public function prePersist(PrePersistEventArgs $args): void
    {
        $entity = $args->getObject();
        if (method_exists($entity, 'setCreatedAt') && $entity->getCreatedAt() === null) {
            $entity->setCreatedAt(new \DateTimeImmutable());
        }
    }
}
```

### Custom domain events

Define an event class (plain object):

```php
namespace App\Event;

use App\Entity\User;

final class UserRegistered
{
    public function __construct(public readonly User $user) {}
}
```

Dispatch:

```php
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

public function __construct(private EventDispatcherInterface $dispatcher) {}

$this->dispatcher->dispatch(new UserRegistered($user));
```

Listen:

```php
#[AsEventListener(event: UserRegistered::class)]
final class SendWelcomeEmailListener
{
    public function __invoke(UserRegistered $event): void { /* … */ }
}
```

Event name defaults to the FQCN — no string registration needed.

### Stopping propagation

```php
$event->stopPropagation();
```

Higher-priority listeners run first; calling `stopPropagation()` prevents
lower-priority listeners on the same event from running.

### Priority

Higher number = runs first. Defaults to 0. Use sparingly — explicit
priorities are hard to reason about as the codebase grows.

## Gotchas

- Listeners are services. They get autowired — type-hint dependencies in
  the constructor.
- A listener that mutates state in `KernelEvents::REQUEST` runs on **every
  request**, including the dev profiler. Filter early (`if (!$event->isMainRequest()) return;`).
- For "send mail after response" use `KernelEvents::TERMINATE` (or, better,
  dispatch a Messenger message and consume async).
- Don't use Doctrine `@HasLifecycleCallbacks` for cross-cutting concerns —
  use `#[AsDoctrineListener]` so logic lives outside entities.
- Stop-propagation flags should be the exception, not the rule.

## Docs

- Event dispatcher: https://symfony.com/doc/7.4/event_dispatcher.html
- Built-in kernel events: https://symfony.com/doc/7.4/reference/events.html
- Doctrine events: https://symfony.com/doc/current/bundles/DoctrineBundle/entity-listeners.html
