# What's new in Symfony 7.4

## Version gate — check before using

These features exist only in **Symfony 7.4 and later** (including all 8.x
releases). Before suggesting or writing code that uses anything on this
page, verify the project's Symfony version:

```bash
composer show symfony/framework-bundle --no-ansi | grep versions
# or, without composer available:
grep '"symfony/framework-bundle"' composer.json
```

(Prefix with `ddev` when the project has `.ddev/config.yaml`.)

- Version **>= 7.4** (or any 8.x) → features below are available.
- Version **< 7.4** (7.3, 7.2, 6.4 LTS, …) → do **not** use them; fall
  back to the standard patterns in the other reference files.

For features that need **8.1+**, see [whats-new-8.1.md](whats-new-8.1.md).

## Console: improved invokable commands

`#[Argument]` / `#[Option]` accept backed enums (auto-converted, clear
errors on invalid values), inputs can be grouped into DTOs, and prompts
can be declared with attributes:

```php
#[AsCommand('app:add-server')]
class AddServerCommand
{
    public function __invoke(
        OutputInterface $output,
        #[Argument, Ask('Enter the cloud region name')] CloudRegion $region, // backed enum
        #[Option] ?ServerSize $size = null,
    ): int {
        return Command::SUCCESS;
    }
}
```

DTO-based input with `#[MapInput]`:

```php
class AddServerInput
{
    #[Argument]
    #[Ask('Enter the cloud region name')]
    public CloudRegion $region;

    #[Option]
    public ?ServerSize $size = null;
}

public function __invoke(#[MapInput] AddServerInput $server): int { /* … */ }
```

`#[Interact]` on a public method handles interactive mode (fill missing
arguments via `SymfonyStyle`). `CommandTester` supports all of this.

## Form: multi-step forms (form flows)

Native multi-step forms via `AbstractFlowType` — each step is a regular
form type, the step name doubles as the active validation group:

```php
use Symfony\Component\Form\Flow\AbstractFlowType;

class UserSignUpType extends AbstractFlowType
{
    public function buildFormFlow(FormFlowBuilderInterface $builder, array $options): void
    {
        $builder->addStep('personal', UserSignUpPersonalType::class);
        $builder->addStep('account', UserSignUpAccountType::class);
        $builder->add('navigator', NavigatorFlowType::class);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => UserSignUp::class,
            'step_property_path' => 'currentStep',
        ]);
    }
}
```

Controller: `$flow->isSubmitted() && $flow->isValid() && $flow->isFinished()`,
render `$flow->getStepForm()`. Navigation types: `NextFlowType`,
`PreviousFlowType`, `FinishFlowType`, `ResetFlowType` (options: `skip`,
`back_to`, `include_if`, `clear_submission`).

## HttpClient: RFC 9111 client-side caching

```yaml
framework:
    cache:
        pools:
            example_cache_pool: { adapter: cache.adapter.redis_tag_aware, tags: true }
    http_client:
        scoped_clients:
            example.client:
                base_uri: 'https://example.com'
                caching:
                    cache_pool: example_cache_pool   # must be tag-aware
                    # shared: true, max_ttl: 3600
```

Responses are cached per HTTP cache headers automatically. Prefer this
over hand-rolled response caching around `HttpClientInterface`.

## Messenger: signed messages

HMAC-sign queued messages with `kernel.secret`; tampered or unsigned
messages throw `InvalidMessageSignatureException` and are discarded:

```php
#[AsMessageHandler(sign: true)]
class SmsNotificationHandler { /* … */ }
```

## Security: voter improvements

- Twig: `access_decision()` / `access_decision_for_user()` return an
  `AccessDecision` with `isGranted()`, votes, and `message` — use to
  explain *why* access was denied in templates.
- Voters receive a `?Vote $vote` argument; attach metadata via
  `$vote->extraData['key'] = …` (usable in custom decision strategies).
- `#[IsGranted('ROLE_ADMIN', methods: ['POST'])]` restricts the check to
  specific HTTP methods.
- New `#[IsSignatureValid]` validates signed URIs on controllers.

## Uid: UUIDv7 by default + test factory

`UuidFactory::create()` now generates UUIDv7 (time-ordered, microsecond
precision). For deterministic tests inject `MockUuidFactory`:

```php
$factory = new MockUuidFactory([
    UuidV4::fromString('11111111-1111-4111-8111-111111111111'),
]);
```

## Validator: Video constraint

`#[Assert\Video]` validates video files (requires ffprobe): `minWidth`,
`maxWidth`, `minHeight`, `maxHeight`, `minRatio`/`maxRatio`,
`allowPortrait`, `allowSquare`, `allowedCodecs: ['h264']`,
`allowedContainers: ['mp4', 'webm']`, plus the usual `maxSize`/`mimeTypes`.

## Validator/Serializer: extend metadata of external classes

Add constraints or serialization metadata to classes you don't own
(vendor DTOs) with attributes instead of XML/YAML files:

```php
#[ExtendsValidationFor(UserRegistration::class)]
class UserRegistrationValidation
{
    #[Assert\NotBlank(groups: ['my_app'])]
    public string $name = '';
}
```

Same idea with `#[ExtendsSerializationFor]` for `#[Groups]`,
`#[SerializedName]`, `#[MaxDepth]`. Property names must match the target
class (checked at container compilation).

## Workflow: weighted transitions + enums

- Transitions accept `weight` on `from`/`to` places — model "N approvals
  before publishing" or fan-in joins.
- Places/markings can be backed enums (`!php/enum App\…\Status::Draft`
  in YAML config).

## Routing / DI attribute improvements

- `#[Route(..., env: ['dev', 'test'])]` — register a route in multiple envs.
- `#[CurrentUser] AdminUser|Customer $user` — union types supported.
- `#[AsEventListener]` methods accept union event types.
- `#[AsDecorator]` is repeatable (decorate several services with one class).
- Simplified `#[Target('foo_package')]` — no type-specific suffix needed
  for asset packages, lock factories, rate limiters.
- `#[AutoconfigureResourceTag('my.tag', ['foo' => 'bar'])]` tags resources.

## HttpFoundation: Request changes (deprecations!)

- `$request->get('key')` is **deprecated** (removed in 8.0). Use
  `$request->attributes->get()`, `$request->query->get()`, or
  `$request->request->get()` explicitly — never write new code with
  `Request::get()`.
- Request body is parsed for `PUT`/`PATCH`/`DELETE`/`QUERY` with form
  payloads (PHP 8.4+).
- HTTP method override of safe methods is deprecated; restrict via
  `framework.allowed_http_method_override: ['PUT', 'DELETE', 'PATCH']`.

## Doctrine: DatePoint column types

With `symfony/clock`'s `DatePoint`, new column types `date_point`
(datetime), `day_point` (date-only), `time_point` (time-only):

```php
#[ORM\Column(type: 'day_point')]
public DatePoint $birthday;
```

## Router: explicit query parameters

Force a parameter into the query string even when it would match a route
placeholder, via the `_query` key:

```php
$urlGenerator->generate('admin_stats', ['siteCode' => 'fr', '_query' => ['siteCode' => 'us']]);
// https://fr.example.com/admin/stats?siteCode=us
```

## Kernel: share directory

`kernel.share_dir` (`APP_SHARE_DIR`, default `var/share`) separates data
shared across servers (cache pools, HTTP cache, SQLite, local storage)
from per-server caches in `var/cache`. App cache pools default to
`%kernel.share_dir%/pools/app`. Point `APP_SHARE_DIR` at the shared mount
in multi-server setups.

## Intl/Form: currency filtering

- `Currencies::forCountry('FR')`, `Currencies::isValidInCountry('CH', 'CHF')`,
  with `legalTender`/`active`/`date` filters.
- `CurrencyType` filters obsolete currencies by default; options
  `legal_tender`, `active_at`, `not_active_at`.

## Configuration direction

- **XML configuration is deprecated** — don't write new XML config.
- A new array-based PHP config format (`App::config([...])` with a
  generated `config/reference.php` for autocompletion) replaces the
  fluent config builders. YAML remains fine and is still the default in
  this skill's examples.

## Smaller items (one-liners)

- Tests: `$client->getSession()` on `WebTestCase` clients to prepare
  session data; `assertEmailAddressNotContains()`; `ClockMock` now also
  mocks `strtotime()`.
- Console: `ConfirmationQuestion::setTimeout(10)`; `messenger:consume
  --exclude-receivers=…`; `security:oidc-token:generate` command.
- Forms: PHP enum properties are auto-guessed as `EnumType`; ARIA
  attributes (`aria-invalid`, `aria-describedby`) added automatically.
- `dd()`/`dump()` output plain text unless the request accepts HTML.
- `#[Assert\Url(protocols: ['*'])]` accepts any RFC 3986 scheme.
- `WebLink\HttpHeaderParser` parses `Link` headers.
- Lock: DynamoDB store (`symfony/amazon-dynamo-db-lock`,
  `dynamodb://default/lock`).
- Serializer: `XmlEncoder::CDATA_WRAPPING_NAME_PATTERN` for per-field
  CDATA.
- Translation: `StaticMessage` — `TranslatableInterface` that skips
  translation.
- Mailer: Microsoft Graph transport.
- `Request::getFormat('application/hal+xml', true)` returns the
  structured-suffix format (`xml`).
- Controller helpers without `AbstractController`: inject closures via
  `#[AutowireMethodOf(ControllerHelper::class)]` (e.g. `render`,
  `redirectToRoute`) for framework-decoupled controllers.

## Docs

- https://symfony.com/blog/symfony-7-4-curated-new-features
- Per-feature posts: https://symfony.com/blog/new-in-symfony-7-4-<feature>
