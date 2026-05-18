# Doctrine — entities, migrations, repositories

## When to use

You need to persist data in a relational DB — create/read/update/delete
entities, write queries, manage schema with migrations.

## What you need

```
composer require orm                    # symfony/orm-pack
composer require migrations             # doctrine/doctrine-migrations-bundle
composer require maker --dev            # symfony/maker-bundle
```

Configure `DATABASE_URL` in `.env.local`. Then:

```sh
bin/console doctrine:database:create
```

(With ddev: `ddev php bin/console …` — applies to every command below.)

## Minimal example — an entity

```php
namespace App\Entity;

use App\Repository\ProductRepository;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Types\UuidType;
use Symfony\Component\Uid\Uuid;

#[ORM\Entity(repositoryClass: ProductRepository::class)]
final class Product
{
    #[ORM\Id]
    #[ORM\Column(type: UuidType::NAME, unique: true)]
    private Uuid $id;

    #[ORM\Column(length: 255)]
    private string $name;

    #[ORM\Column(type: 'integer')]
    private int $priceCents;

    #[ORM\Column]
    private \DateTimeImmutable $createdAt;

    public function __construct(string $name, int $priceCents)
    {
        $this->id = Uuid::v7();
        $this->name = $name;
        $this->priceCents = $priceCents;
        $this->createdAt = new \DateTimeImmutable();
    }

    public function getId(): Uuid { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getPriceCents(): int { return $this->priceCents; }
    public function getCreatedAt(): \DateTimeImmutable { return $this->createdAt; }
}
```

## Workflow

### Create an entity

```
bin/console make:entity Product
```

Interactive — adds fields one at a time, generates the class and (empty)
repository.

### Generate a migration after schema changes

```
bin/console make:migration
```

**Always use `make:migration`** (don't write migrations by hand). Review
the generated SQL, then:

```
bin/console doctrine:migrations:migrate
```

In tests:

```
bin/console doctrine:migrations:migrate --env=test
# or
bin/console doctrine:schema:create --env=test  # without migrations
```

### Repositories

```php
namespace App\Repository;

use App\Entity\Product;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

/**
 * @extends ServiceEntityRepository<Product>
 */
final class ProductRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Product::class);
    }

    /** @return Product[] */
    public function findCheaperThan(int $cents): array
    {
        return $this->createQueryBuilder('p')
            ->where('p.priceCents < :cents')
            ->setParameter('cents', $cents)
            ->orderBy('p.priceCents', 'ASC')
            ->getQuery()
            ->getResult();
    }
}
```

Inject by class — autowiring handles it:

```php
public function __construct(private ProductRepository $products) {}

$cheap = $this->products->findCheaperThan(1000);
```

### Persisting

```php
use Doctrine\ORM\EntityManagerInterface;

public function __construct(private EntityManagerInterface $em) {}

public function create(string $name, int $price): Product
{
    $product = new Product($name, $price);
    $this->em->persist($product);
    $this->em->flush();
    return $product;
}
```

For updates, just mutate the entity and `flush()` — no `persist()` needed.

For deletes:

```php
$this->em->remove($product);
$this->em->flush();
```

## Common patterns

### Relations

```php
#[ORM\ManyToOne(targetEntity: Category::class, inversedBy: 'products')]
#[ORM\JoinColumn(nullable: false)]
private Category $category;

// inverse side:
#[ORM\OneToMany(mappedBy: 'category', targetEntity: Product::class)]
private Collection $products;
```

### Enum columns (PHP 8.1+ enums)

```php
enum OrderStatus: string {
    case Pending = 'pending';
    case Paid = 'paid';
    case Shipped = 'shipped';
}

#[ORM\Column(enumType: OrderStatus::class)]
private OrderStatus $status;
```

### Embeddables

```php
#[ORM\Embedded(class: Money::class)]
private Money $price;
```

### Lifecycle callbacks

Prefer event listeners with `#[AsDoctrineListener]` over entity-level
`@HasLifecycleCallbacks` — keeps entities clean.

```php
use Doctrine\Bundle\DoctrineBundle\Attribute\AsDoctrineListener;
use Doctrine\ORM\Event\PrePersistEventArgs;
use Doctrine\ORM\Events;

#[AsDoctrineListener(event: Events::prePersist)]
final class TimestampableListener
{
    public function prePersist(PrePersistEventArgs $args): void { /* … */ }
}
```

### Pagination — `Paginator`

```php
use Doctrine\ORM\Tools\Pagination\Paginator;

$qb = $repo->createQueryBuilder('p')
    ->orderBy('p.createdAt', 'DESC')
    ->setFirstResult($offset)
    ->setMaxResults($limit);

$paginator = new Paginator($qb);
$total = count($paginator);
$items = iterator_to_array($paginator);
```

### Bulk operations — iterate, batch flush

```php
$batchSize = 100;
$i = 0;
foreach ($iterable as $row) {
    $em->persist(new Entity($row));
    if (++$i % $batchSize === 0) {
        $em->flush();
        $em->clear(); // detach to free memory
    }
}
$em->flush();
```

## Gotchas

- **Always run `make:migration` after schema changes.** Never edit the DB
  manually in dev or write a migration by hand.
- `doctrine:schema:update --force` is for sandboxes only. Migrations are
  the source of truth in any real environment.
- `flush()` is per-EntityManager. Don't call it in tight loops without
  `clear()` — Unit of Work grows unbounded.
- Avoid the `Repository` annotation pattern of injecting and then using
  `$em->getRepository(Foo::class)` — inject the specific
  `FooRepository` instead.
- Lazy-loading proxies + serialization can produce surprises. For DTOs
  reaching the wire, prefer the serializer with explicit groups or
  hydrate plain arrays via DQL `NEW App\Dto\X(...)` syntax.
- Don't store `\DateTime` (mutable) — always `\DateTimeImmutable`.

## Docs

- Doctrine + Symfony: https://symfony.com/doc/7.4/doctrine.html
- Migrations bundle: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
- ORM mapping reference: https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/attributes-reference.html
