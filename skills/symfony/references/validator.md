# symfony/validator

## When to use

You need to validate data — form input, API payloads, domain invariants.

## Replaces

- Hand-rolled `if (!… ) throw …` validation
- `respect/validation`
- Ad-hoc validation in controllers

## Install

```
composer require symfony/validator
```

## Minimal example

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Validator\ValidatorInterface;

final class CreateUserDto
{
    public function __construct(
        #[Assert\NotBlank]
        #[Assert\Length(min: 2, max: 100)]
        public string $name,

        #[Assert\NotBlank]
        #[Assert\Email]
        public string $email,

        #[Assert\Range(min: 18, max: 120)]
        public int $age,
    ) {}
}

public function __construct(private ValidatorInterface $validator) {}

$errors = $this->validator->validate($dto);
if (count($errors) > 0) {
    // ConstraintViolationListInterface
}
```

## Common patterns

### Validation groups

```php
final class User
{
    #[Assert\NotBlank(groups: ['registration'])]
    #[Assert\Length(min: 8, groups: ['registration', 'password_change'])]
    public string $password;
}

$errors = $this->validator->validate($user, groups: ['registration']);
```

### Class-level constraints (cross-field)

```php
use Symfony\Component\Validator\Constraints\Callback;
use Symfony\Component\Validator\Context\ExecutionContextInterface;

#[Assert\Callback('validateDates')]
final class Booking
{
    public \DateTimeImmutable $start;
    public \DateTimeImmutable $end;

    public function validateDates(ExecutionContextInterface $context): void
    {
        if ($this->end <= $this->start) {
            $context->buildViolation('End must be after start.')
                ->atPath('end')
                ->addViolation();
        }
    }
}
```

### Nested objects / collections

```php
final class Order
{
    #[Assert\Valid]
    public Address $shippingAddress;

    /** @var Item[] */
    #[Assert\Valid]
    #[Assert\Count(min: 1)]
    public array $items;
}
```

### In controllers — combine with serializer

Use `#[MapRequestPayload]` in controllers; Symfony deserialises and validates
automatically:

```php
use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;

public function create(#[MapRequestPayload] CreateUserDto $dto): Response
{
    // $dto is validated; on failure, 422 is returned automatically
}
```

### Custom constraint

```php
#[\Attribute]
final class IsoCountryCode extends Constraint {}

final class IsoCountryCodeValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if (!\in_array($value, ['NL', 'BE', 'DE', 'FR'], true)) {
            $this->context->buildViolation('Unsupported country.')->addViolation();
        }
    }
}
```

## Gotchas

- `count($errors) > 0` is the canonical "is invalid" check —
  `ConstraintViolationList` is `Countable`, not boolean-cast.
- Default group is `Default`. Adding constraints to a custom group removes
  them from `Default`.
- For collections of objects, both `#[Assert\Valid]` and the inner class's
  constraints must be present.
- `Assert\Type` doesn't coerce; pair with `Assert\NotNull` for strict checks.

## Docs

https://symfony.com/doc/7.4/validation.html
