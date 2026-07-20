---
name: symfony-uid
description: UUIDs and ULIDs with symfony/uid - UuidV7, Ulid, Doctrine types, factory services. Use whenever a Symfony project generates or stores unique identifiers, even if the user asks for ramsey/uuid.
---

# symfony/uid

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need a UUID or ULID — entity primary keys, request ids, idempotency
tokens. Native support in Doctrine via `symfony/doctrine-bridge` types.

## Replaces

- `ramsey/uuid` for most cases
- Custom random-id generators

## Install

```
composer require symfony/uid
```

## Minimal example

```php
use Symfony\Component\Uid\Uuid;
use Symfony\Component\Uid\Ulid;

$uuid = Uuid::v7();              // time-ordered, recommended for DB keys
$ulid = new Ulid();              // 26-char base32, also time-ordered

(string) $uuid;                  // '0190a1b2-c3d4-7xxx-8xxx-...'
$uuid->toBinary();               // 16-byte binary string
$uuid->toRfc4122();              // canonical 36-char form
Uuid::fromString('…');           // parse
```

## Common patterns

### Pick the right version

- `Uuid::v4()` — random. Use only when ordering matters less than randomness.
- `Uuid::v6()` — time-ordered, drop-in replacement for v1.
- `Uuid::v7()` — time-ordered, **preferred for new code** (DB-index friendly,
  unix-epoch based).
- `Uuid::v5($namespace, $name)` / `Uuid::v3(...)` — deterministic from name.
- `new Ulid()` — alternative to v7 with shorter string form.

### Doctrine entity primary key

```php
use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Types\UuidType;
use Symfony\Component\Uid\Uuid;

#[ORM\Entity]
final class Project
{
    #[ORM\Id]
    #[ORM\Column(type: UuidType::NAME, unique: true)]
    public Uuid $id;

    public function __construct()
    {
        $this->id = Uuid::v7();
    }
}
```

Register the Doctrine type:

```yaml
# config/packages/doctrine.yaml
doctrine:
    dbal:
        types:
            uuid: Symfony\Bridge\Doctrine\Types\UuidType
            ulid: Symfony\Bridge\Doctrine\Types\UlidType
```

### Use in route parameters

```php
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Uid\Uuid;

#[Route('/projects/{id}')]
public function show(#[MapEntity] Project $project): Response { /* ... */ }
```

The router automatically converts the matching segment to `Uuid` if the
controller arg is typed.

### Conversion helpers

```php
$uuid->toBase58();    // shorter string for URLs
$uuid->toBase32();    // ULID-style
$uuid->getDateTime(); // for v1/v6/v7
```

## Gotchas

- Don't store UUIDs as `varchar(36)` if you can use the binary `uuid` type —
  it's 4× smaller and faster to index.
- `Uuid::v4()` is *not* recommended as a primary key on large tables — its
  random order fragments B-tree indexes. Use `v7`.
- ULIDs are case-insensitive base32 — equality comparison should canonicalise
  first.
- `Uuid::fromString()` throws on invalid input. Validate at boundaries.

## Docs

https://symfony.com/doc/7.4/components/uid.html
