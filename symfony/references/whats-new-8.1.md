# What's new in Symfony 8.1

## Version gate — check before using

These features exist only in **Symfony 8.1 and later**. Before suggesting
or writing code that uses anything on this page, verify the project's
Symfony version:

```bash
composer show symfony/framework-bundle --no-ansi | grep versions
# or, without composer available:
grep '"symfony/framework-bundle"' composer.json
```

(Prefix with `ddev` when the project has `.ddev/config.yaml`.)

- Version **>= 8.1** → features below are available.
- Version **7.4 / 8.0** → do **not** use them; the 7.4 feature set in
  [whats-new-7.4.md](whats-new-7.4.md) still applies.
- Version **< 7.4** → neither page applies.

## Console: method-based commands

Group related commands in one class — `#[AsCommand]` on methods, shared
constructor dependencies:

```php
class UserCommands
{
    public function __construct(private UserRepository $users) {}

    #[AsCommand('app:user:create', description: 'Creates a new user')]
    public function create(OutputInterface $output): int
    {
        return Command::SUCCESS;
    }

    #[AsCommand('app:user:delete', description: 'Deletes an existing user')]
    public function delete(OutputInterface $output): int
    {
        return Command::SUCCESS;
    }
}
```

Autoconfiguration registers each method as a command. Test with
`new CommandTester($instance->create(...))`.

## Console: argument resolvers + DI in `__invoke()`

Controller-style value resolution now works in commands:

```php
#[AsCommand('app:report:generate')]
final class GenerateReportCommand
{
    public function __invoke(
        ReportGenerator $reports,                                  // service, no constructor needed
        #[Argument, MapEntity] User $user,                         // entity by id
        #[Option, MapDateTime(format: 'Y-m-d')] \DateTimeInterface $date,
        #[Autowire('%kernel.environment%')] string $env,
    ): int { /* … */ }
}
```

Backed enums, UUID/ULID resolve out of the box; custom resolvers
implement `ValueResolverInterface`.

## Console: richer interactive input

- `#[AskChoice('Select a role', ['admin', 'editor'])]` — declarative
  choice prompt; choices auto-derived from a `BackedEnum` type; `array`
  type allows multi-select.
- `#[Ask('Enter your email:', constraints: [new Assert\NotBlank(), new Assert\Email()])]`
  — validate answers with Validator constraints.
- `#[MapInput(validationGroups: ['registration'])]` — validate whole
  input DTOs (`InputValidationFailedException` on failure).
- `InputFile` type accepts pasted images or file paths.
- `ProgressBar::pauseAll()` / `resumeAll()` around prompts.

## Console: testing improvements

```php
// KernelTestCase shortcut: boots kernel, finds command, wraps tester
$commandTester = static::runCommand('app:create-user', ['username' => '…']);
$commandTester->assertCommandIsSuccessful();
$commandTester->assertCommandFailed();      // Command::FAILURE
$commandTester->assertCommandIsInvalid();   // Command::INVALID
```

`CommandTester::run()` returns an `ExecutionResult` (`getOutput()`,
`getErrorOutput()`, `statusCode`) and accepts `interactiveInputs: [...]`.
`ConsoleAssertionsTrait` adds `assertIsSuccessful()` /
`assertResultEquals()`.

## HttpKernel: `#[Serialize]` controller attribute

Return objects directly from controllers; the kernel serializes them
into the response (format negotiated from the request, JSON default):

```php
use Symfony\Component\HttpKernel\Attribute\Serialize;

final readonly class GetUserController
{
    #[Serialize(code: 201, context: [DateTimeNormalizer::FORMAT_KEY => 'd.m.Y'])]
    public function __invoke(): User
    {
        return new User(1, 'Jane Smith');
    }
}
```

Prefer this over manual `$serializer->serialize()` + `JsonResponse` in
API controllers.

## HttpKernel: improved `#[Cache]` attribute

`lastModified`/`etag` accept closures (better than expression strings for
static analysis), expressions expose `request` and `args`, and `if` makes
the attribute conditional and repeatable:

```php
#[Cache(
    public: true,
    maxage: 3600,
    lastModified: static fn (array $args, Request $request): \DateTimeInterface => $args['post']->getUpdatedAt(),
    if: static fn (array $args, Request $request): bool => !$request->query->has('preview'),
)]
public function show(Post $post): Response { /* … */ }
```

## RateLimiter: `#[RateLimit]` attribute + calendar windows

```php
#[RateLimit('api')]                                          // limiter name from framework.rate_limiter
#[RateLimit('per_account', key: new Expression('request.request.get("email")'))]
public function index(): JsonResponse { /* … */ }
```

Returns `429` with `Retry-After` automatically; default key is
IP + method + path; supports `methods:`, `tokens:`, class-level use, and
stacking. New `anchor_at` config option aligns `fixed_window` limiters to
calendar resets (billing cycles):

```yaml
framework:
    rate_limiter:
        api_quota:
            policy: 'fixed_window'
            limit: 10000
            interval: '1 month'
            anchor_at: '2026-01-05 00:00:00 UTC'
```

## HttpFoundation: request payload mapping

- `#[MapRequestPayload]` maps `multipart/form-data` including
  `UploadedFile` properties inside DTOs.
- Variadic mapping: `#[MapRequestPayload] Price ...$prices` unpacks a
  JSON array into DTO instances.
- `#[MapQueryString(mapWhenEmpty: true)]` runs denormalization on empty
  input.
- `validationGroups` accepts an `Expression` or `Closure` with access to
  resolved controller args:
  `#[MapRequestPayload(validationGroups: [new Expression('args["user"].getType()')])]`.

## Validator

- `#[Assert\Xml]` — well-formed XML, optional
  `schemaPath: 'config/schemas/report.xsd'`.
- `GreaterThan`/`LessThan`/`Range` (and friends) accept a
  `ClockInterface` so relative dates resolve against `MockClock` in tests.
- `ValidatorBuilder::enablePropertyMetadataExistenceCheck()` throws on
  validating a property that doesn't exist (catches typos).

## Messenger

- `messenger:consume --fetch-size=8` — batch-fetch from SQS/Redis/
  Doctrine/AMQP.
- `--no-reset=100` — reset container services every N messages.
- `#[AsMessage(serializedTypeName: 'crawler.vectorization_finished')]` —
  decouple the wire type name from the FQCN.
- `AmqpPriorityStamp(5)` — per-message AMQP priority.
- Batch handlers: `getIdleTimeout(): ?float` flushes partial batches.
- AMQP: disable default queue binding with `queues: false` (pub-only
  transports); Redis Cluster via DSN `?redis_cluster=true`; Redis
  receiver implements `ListableReceiverInterface` (`all()`, `find()`).
- Decode failures now go through normal retry/failure handling.

## DependencyInjection

- Env vars as closures/stringables — lazy re-reads in long-running
  workers: `#[Autowire(env: 'DB_URL')] private \Closure $dbUrl` (YAML:
  `!env_closure`).
- Decorate every service with a tag: `#[AsTagDecorator('app.handler')]`
  or `decorates_tag:` in YAML; service `stack`s can now decorate.
- `#[AsAlias(StorageInterface::class, target: 'image')]` pairs with
  `#[Target('image')]` for named autowiring.
- `#[AsTaggedItem(index: 'json', priority: 10)]` replaces the deprecated
  `getDefaultIndexName()`/`getDefaultPriority()` static methods — also
  sets voter priority.
- `$configurator->import('services/*.php', exclude: ['services/legacy/*'])`.
- Env var names may contain dots: `%env(DATABASE.PRIMARY.URL)%`.

## HttpClient

- `extra.use_persistent_connections: true` (cURL) — reuse connections in
  workers.
- `NoPrivateNetworkHttpClient` third argument allow-lists specific
  private IPs (SSRF protection with exceptions).
- `max_connect_duration` request option caps connection setup time.
- Per-client mocking in tests: `framework.http_client.mock_response_factory`
  globally and per scoped client (`false` to opt out).
- `DnsResolvingHttpClient` decorator for custom hostname→IP resolution.
- Caching client: `max_ttl` per scoped client (default 86400).

## VarExporter: DeepCloner

Deep-clone object graphs 4–15× faster than `unserialize(serialize())`:

```php
use Symfony\Component\VarExporter\DeepCloner;

$clone = DeepCloner::deepClone($original);          // one-off
$cloner = new DeepCloner($prototype);               // reusable
$child = (new DeepCloner($def))->cloneAs(ChildDefinition::class);
```

`deepclone_hydrate(User::class, ['name' => 'Alice'])` replaces the
deprecated `Instantiator`/`Hydrator`.

## ObjectMapper

- `#[Map(source: Quote::class)]` on the *target* class keeps domain
  objects free of mapping metadata; `$mapper->map($quote)` needs no
  target argument.
- `#[Map(if: new IsNotNull())]` skips null source values (PATCH DTOs).
- `#[Map(transform: new MapCollection(targetClass: LineItem::class))]`
  maps collection items to a class.
- `TargetClass`/`SourceClass` conditions accept arrays; invalid
  `transform`/`if` callables now throw `NoSuchCallableException`.

## Form

- Form flows: `$this->createFormFlowBuilder(new User())->addStep(...)`
  shortcut; `NavigatorFlowType` `with_reset` option; Twig helpers
  `form_flow_step_index()`, `form_flow_total_steps()`,
  `form_flow_can_move_back()`.
- `DateType` choice widget: `labels` option for year/month/day.
- `EntityType`: `uid_format: 'base58'` for UUID/ULID identifiers.
- Unchecked checkboxes submit as `false` automatically (POST/PUT).
- New daisyUI form theme: `daisyui_5_layout.html.twig`.

## Translation

- `framework.enabled_locales` accepts `%env()%` values (empties filtered).
- `LocaleFallbackProvider` computes fallback chains
  (`es_AR` → `['es_419', 'es', 'en']`).
- XLIFF 2.1/2.2 accepted transparently.

## JsonStreamer / JsonPath

- `ValueObjectTransformerInterface` serializes value objects as scalars;
  `DateInterval`/`DateTimeZone` handled natively; `date_time_timezone`
  and `date_interval_format` options.
- `framework.json_streamer.default_options` sets app-wide defaults.
- `#[AsJsonPathFunction('upper')]` registers custom JsonPath functions:
  `$crawler->find('$.items[?upper(@.title) == "HELLO"]')`.

## HttpKernel: dynamic controller attributes

Controller attributes live in the `_controller_attributes` request
attribute; listeners can read/override them via
`ControllerEvent::getAttributes()` / `setController()`, and a dedicated
event fires per attribute (`kernel.controller_arguments.{AttributeFQCN}`,
listener receives `ControllerAttributeEvent`) — the clean way to build
custom attribute-driven behavior (the `#[RateLimit]` attribute is built
on this).

## Skipped as niche

HTTP-less kernels (`AbstractKernel` in DependencyInjection), Guzzle
handler bridge, XLIFF PGS module, terminal taskbar progress (automatic).
Look them up in the 8.1 blog series if a task genuinely calls for them.

## Docs

- https://symfony.com/blog/symfony-8-1-curated-new-features
- Per-feature posts: https://symfony.com/blog/new-in-symfony-8-1-<feature>
