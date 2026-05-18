# symfony/serializer

## When to use

You need to convert objects to/from JSON, XML, CSV, or YAML — REST APIs,
queue payloads, config dumps.

## Replaces

- JMS Serializer (`jms/serializer`)
- `json_encode`/`json_decode` + manual object hydration
- Hand-rolled `toArray()` / `fromArray()` on every DTO

## Install

```
composer require symfony/serializer-pack
```

The pack pulls in `symfony/serializer`, `symfony/property-info`,
`symfony/property-access`, and metadata reading.

## Minimal example

```php
use Symfony\Component\Serializer\SerializerInterface;

final class UserDto
{
    public function __construct(
        public string $name,
        public int $age,
    ) {}
}

public function __construct(private SerializerInterface $serializer) {}

$json = $this->serializer->serialize(new UserDto('Alice', 30), 'json');
$dto  = $this->serializer->deserialize($json, UserDto::class, 'json');
```

## Common patterns

### Serialization groups

```php
use Symfony\Component\Serializer\Attribute\Groups;

final class User
{
    public function __construct(
        #[Groups(['public', 'admin'])] public string $name,
        #[Groups(['admin'])]            public string $email,
    ) {}
}

$publicJson = $this->serializer->serialize($user, 'json', ['groups' => ['public']]);
```

### Date format

```php
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

$this->serializer->serialize($obj, 'json', [
    DateTimeNormalizer::FORMAT_KEY => \DateTimeInterface::ATOM,
]);
```

### Renaming fields

```php
use Symfony\Component\Serializer\Attribute\SerializedName;

final class Product
{
    public function __construct(
        #[SerializedName('product_id')] public int $id,
    ) {}
}
```

### Ignoring fields

```php
use Symfony\Component\Serializer\Attribute\Ignore;

final class User
{
    public string $email;
    #[Ignore] public string $passwordHash;
}
```

### Populate existing object (PATCH semantics)

```php
$this->serializer->deserialize($json, User::class, 'json', [
    AbstractNormalizer::OBJECT_TO_POPULATE => $existingUser,
]);
```

### CSV / XML

```php
$csv = $this->serializer->serialize($rows, 'csv');
$xml = $this->serializer->serialize($obj, 'xml');
```

## Gotchas

- For deserialising into typed collections (`User[]`), use `'User[]'` as the
  type or pass a custom context. The serializer needs `PropertyInfo` to
  figure out generics.
- The `denormalize` extra strictness option (`ALLOW_EXTRA_ATTRIBUTES => false`)
  is off by default — unknown keys are silently ignored. Turn it on for
  inbound API validation.
- For complex DTOs prefer attributes over YAML/XML metadata — easier to keep
  in sync with the class.
- The `SerializerInterface` is the encoder. For one-step
  array-to-object conversion without a format, use `DenormalizerInterface`.

## Docs

https://symfony.com/doc/7.4/serializer.html
