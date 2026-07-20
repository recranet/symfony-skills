---
name: symfony-workflow
description: State machines with symfony/workflow - places, transitions, guards, events, marking stores. Use whenever a Symfony model has a status or state field with rules about allowed transitions, even if the user proposes if/else status checks.
---

# symfony/workflow

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need to model the lifecycle of an entity with discrete states and
transitions — order status, document approval, subscription lifecycle.

## Replaces

- Hand-rolled state machines (`if ($order->status === 'pending') { ... }`)
- Boolean flag soup (`is_paid`, `is_shipped`, `is_cancelled`)
- Third-party state-machine packages

## Install

```
composer require symfony/workflow
```

## Minimal example

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        order:
            type: 'state_machine'
            marking_store:
                type: 'method'
                property: 'status'
            supports:
                - App\Entity\Order
            initial_marking: 'cart'
            places:
                - cart
                - placed
                - paid
                - shipped
                - cancelled
            transitions:
                place:
                    from: cart
                    to: placed
                pay:
                    from: placed
                    to: paid
                ship:
                    from: paid
                    to: shipped
                cancel:
                    from: [cart, placed, paid]
                    to: cancelled
```

```php
use Symfony\Component\Workflow\WorkflowInterface;

public function __construct(private WorkflowInterface $orderStateMachine) {}

public function payOrder(Order $order): void
{
    if (!$this->orderStateMachine->can($order, 'pay')) {
        throw new \DomainException('Order cannot be paid right now');
    }
    $this->orderStateMachine->apply($order, 'pay');
    // status property is now 'paid'
}
```

Autowire by `<workflowName>StateMachine` or `<workflowName>Workflow`.

## Common patterns

### `workflow` vs `state_machine`

- `state_machine` — exactly one current place. Use for entity lifecycles
  (orders, invoices, tickets).
- `workflow` — multiple concurrent places (Petri nets). Use for parallel
  approval flows.

### Guards (block transitions conditionally)

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Workflow\Event\GuardEvent;

final class OrderGuardListener
{
    #[AsEventListener('workflow.order.guard.ship')]
    public function blockUnpaid(GuardEvent $event): void
    {
        $order = $event->getSubject();
        if (!$order->isFullyPaid()) {
            $event->setBlocked(true, 'Order is not fully paid');
        }
    }
}
```

### React to transitions

```php
#[AsEventListener('workflow.order.completed.pay')]
public function onPaid(CompletedEvent $event): void
{
    $this->mailer->send(...);
}
```

Event names: `workflow.<name>.<event>.<transition>` and
`workflow.<name>.<event>` for the all-transitions variant. Events fired
include `guard`, `leave`, `transition`, `enter`, `entered`, `completed`,
`announce`.

### Metadata on places/transitions

```yaml
workflows:
    order:
        # ...
        places:
            placed:
                metadata:
                    title: 'Order placed'
        transitions:
            ship:
                from: paid
                to: shipped
                metadata:
                    title: 'Ship the order'
                    requires_role: 'ROLE_LOGISTICS'
```

Access via `$workflow->getMetadataStore()->getMetadata(...)`.

### Dump the diagram

```
php bin/console workflow:dump order | dot -Tpng > order.png
```

## Gotchas

- `apply()` throws on disallowed transitions — gate with `can()` first or
  catch `NotEnabledTransition`.
- The marking property must be writable for `marking_store: method`. For
  enum-typed properties use `marking_store: method` with matching getter/
  setter; for simple string properties Symfony 7.4 handles it automatically.
- Workflows are stateless services — the *entity* holds the marking, not the
  workflow.
- Don't mutate the entity inside guard listeners.

## Docs

https://symfony.com/doc/7.4/workflow.html
