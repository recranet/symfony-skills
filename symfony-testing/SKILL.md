---
name: symfony-testing
description: Testing Symfony apps - KernelTestCase, WebTestCase, Panther browser tests, fixtures, in-memory transports, mocking services. Use whenever writing or fixing unit, integration, functional, or end-to-end tests in a Symfony project.
---

# Testing

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You're writing tests for a Symfony 7.4+ project — unit tests, service tests
that need the container, controller/HTTP tests, browser tests.

## What you need

```
composer require test --dev      # symfony/test-pack (phpunit, browser-kit, ...)
composer require panther --dev   # only for real-browser tests
```

`phpunit.xml.dist` lives at the project root. Tests live in `tests/`
(PSR-4 namespace `App\Tests\`).

Run:

```
vendor/bin/phpunit
# or
bin/phpunit
# or
ddev php bin/phpunit
```

## Test types

| Type | Base class | Boots kernel? | Use for |
|------|-----------|---------------|---------|
| Unit | `\PHPUnit\Framework\TestCase` | No | Pure logic, value objects |
| Service / integration | `KernelTestCase` | Yes | Services that need DI, DB, etc. |
| HTTP / functional | `WebTestCase` | Yes (+ test client) | Controllers, routes |
| Browser | `PantherTestCase` | Yes (+ real browser) | JS-heavy flows |

Always pick the lightest type that proves the behaviour.

## Minimal examples

### Unit

```php
namespace App\Tests\Money;

use App\Money\Money;
use PHPUnit\Framework\TestCase;

final class MoneyTest extends TestCase
{
    public function testAdd(): void
    {
        $sum = (new Money(100))->add(new Money(50));
        $this->assertSame(150, $sum->getCents());
    }
}
```

### Service (with container)

```php
namespace App\Tests\Service;

use App\Service\PriceCalculator;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

final class PriceCalculatorTest extends KernelTestCase
{
    public function testCalculate(): void
    {
        self::bootKernel();
        $calc = self::getContainer()->get(PriceCalculator::class);

        $this->assertSame(110, $calc->withTax(100, 0.1));
    }
}
```

`self::getContainer()` returns a special test container that exposes
private services too — fine in tests, never in prod code.

### HTTP / functional

```php
namespace App\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

final class HealthControllerTest extends WebTestCase
{
    public function testHealth(): void
    {
        $client = static::createClient();
        $client->request('GET', '/health');

        $this->assertResponseIsSuccessful();
        $this->assertJsonStringEqualsJsonString(
            '{"ok":true}',
            $client->getResponse()->getContent()
        );
    }
}
```

Built-in assertions: `assertResponseIsSuccessful`, `assertResponseRedirects`,
`assertResponseHeaderSame`, `assertSelectorExists`, `assertSelectorTextContains`,
`assertRouteSame`, `assertResponseStatusCodeSame`, …

### Functional — log in a user

```php
$user = $this->getContainer()->get(UserRepository::class)->findOneBy(['email' => 'a@b']);
$client->loginUser($user);
$client->request('GET', '/profile');
```

For firewalls other than `main`:

```php
$client->loginUser($user, 'api');
```

### Browser (Panther)

```php
use Symfony\Component\Panther\PantherTestCase;

final class SignupTest extends PantherTestCase
{
    public function testSignupFlow(): void
    {
        $client = self::createPantherClient();
        $client->request('GET', '/signup');
        $client->submitForm('Sign up', ['email' => 'a@b.com', 'password' => 'hunter2']);
        $this->assertSelectorTextContains('.flash', 'Welcome');
    }
}
```

Use sparingly — much slower than `WebTestCase`. Only for flows requiring a
real JS runtime.

## Common patterns

### Database isolation

Two common approaches:

**Per-test transaction rollback** (fast):

```php
use Doctrine\ORM\EntityManagerInterface;

protected function setUp(): void
{
    parent::setUp();
    self::bootKernel();
    $this->em = self::getContainer()->get(EntityManagerInterface::class);
    $this->em->beginTransaction();
}

protected function tearDown(): void
{
    $this->em->rollback();
    parent::tearDown();
}
```

**Fresh schema per test** (slow but safe):

```
bin/console doctrine:database:create --env=test --if-not-exists
bin/console doctrine:migrations:migrate --env=test --no-interaction
```

### Fixtures

```
composer require orm-fixtures --dev
bin/console doctrine:fixtures:load --env=test --no-interaction
```

### Mock the clock

```php
use Symfony\Component\Clock\Test\ClockSensitiveTrait;

final class SubscriptionTest extends KernelTestCase
{
    use ClockSensitiveTrait;

    public function testExpiry(): void
    {
        $clock = self::mockTime('2026-01-01 00:00:00');
        // …
        $clock->sleep(86400 * 31);
        $this->assertTrue($subscription->isExpired());
    }
}
```

### Mock the HTTP client

```php
use Symfony\Component\HttpClient\MockHttpClient;
use Symfony\Component\HttpClient\Response\MockResponse;

self::getContainer()->set(HttpClientInterface::class, new MockHttpClient([
    new MockResponse(json_encode(['ok' => true])),
]));
```

### Test a console command

```php
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Tester\CommandTester;

$app = new Application(self::bootKernel());
$tester = new CommandTester($app->find('app:rebuild-search-index'));
$tester->execute([]);

$this->assertSame(0, $tester->getStatusCode());
$this->assertStringContainsString('Indexed', $tester->getDisplay());
```

### Test Messenger handlers

Route messages to `in-memory://` in test env:

```yaml
# config/packages/test/messenger.yaml
framework:
    messenger:
        transports:
            async: 'in-memory://'
```

Then assert on the in-memory transport's collection.

## Gotchas

- `WebTestCase::createClient()` boots a new kernel — call once per test or
  share with `static`.
- `self::getContainer()` is the *test* container — its `get()` works for
  private services too, but it's a different instance than production.
- After modifying `config/services.yaml`, the test cache must be cleared:
  `bin/console cache:clear --env=test`.
- `loginUser()` writes to the session — for stateless firewalls use a
  token-based mechanism instead.
- Don't share state between tests via static properties. PHPUnit doesn't
  reset them.
- Panther starts a real browser — install `chromedriver` or `geckodriver`,
  and don't run Panther tests in CI without one.

## Docs

- Testing: https://symfony.com/doc/7.4/testing.html
- DOM Crawler: https://symfony.com/doc/7.4/components/dom_crawler.html
- Panther: https://github.com/symfony/panther
- HTTP client mocking: https://symfony.com/doc/7.4/http_client.html#testing-http-clients-and-responses
