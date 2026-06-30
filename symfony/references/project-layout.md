# Project layout & tooling

## When to use

You're orienting in a Symfony 7.4+ project — figuring out where files go,
how to run commands, and which conventions apply.

## Directory layout

```
.
├── bin/console               # CLI entrypoint
├── config/
│   ├── bundles.php           # enabled bundles per env
│   ├── packages/             # bundle configuration
│   │   ├── framework.yaml
│   │   ├── doctrine.yaml
│   │   ├── security.yaml
│   │   ├── messenger.yaml
│   │   └── …
│   ├── routes/               # imports
│   ├── routes.yaml
│   ├── services.yaml         # service container config
│   └── preload.php
├── migrations/               # Doctrine migrations
├── public/
│   └── index.php             # web entrypoint
├── src/                      # PSR-4: App\
│   ├── Controller/
│   ├── Entity/
│   ├── Repository/
│   ├── Form/
│   ├── Security/
│   ├── EventListener/
│   ├── EventSubscriber/
│   ├── Command/
│   ├── Message/
│   ├── MessageHandler/
│   ├── Service/
│   ├── DataFixtures/
│   ├── Kernel.php
│   └── …
├── templates/                # Twig
├── tests/                    # PSR-4: App\Tests\
├── translations/
├── var/
│   ├── cache/
│   └── log/
├── vendor/
├── .env
├── .env.local                # gitignored, dev overrides
├── .env.test
├── composer.json
├── symfony.lock              # Flex
└── .ddev/                    # present when using ddev
```

## Tooling — ddev when present

The presence of `.ddev/config.yaml` means commands run inside the ddev
container:

```sh
ddev php bin/console make:migration
ddev php bin/console doctrine:migrations:migrate
ddev composer require lock
ddev composer install
ddev php bin/console cache:clear
```

Without ddev, run the same commands directly on the host:

```sh
php bin/console make:migration
composer require lock
```

If the Symfony CLI is installed, `symfony console …` and `symfony composer
…` are interchangeable equivalents.

## Conventions to rely on

In `config/services.yaml` you'll typically see:

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        public: false

    App\:
        resource: '../src/'
        exclude:
            - '../src/DependencyInjection/'
            - '../src/Entity/'
            - '../src/Kernel.php'
```

What this means:

- **Autowiring**: type-hint a service in your constructor and the container
  injects it. No manual config needed.
- **Autoconfigure**: attributes like `#[AsCommand]`,
  `#[AsMessageHandler]`, `#[AsEventListener]` register the service
  correctly without explicit tags.
- **Private services**: don't fetch services from the container directly;
  inject them.
- **Class = service**: any class under `src/` is auto-registered.

## Environment variables

- `.env` — defaults, committed.
- `.env.local` — local overrides, not committed.
- `.env.test`, `.env.test.local` — test environment.
- Access in PHP via `$_ENV['NAME']` or, in config, `'%env(NAME)%'` /
  `'%env(string:NAME)%'` / `'%env(int:NAME)%'`.

Avoid reading `getenv()` directly; use the framework's env var processors.

## `make:*` recipes (MakerBundle)

When installed (`composer require maker --dev`):

```
make:controller        make:entity            make:migration
make:command            make:form              make:user
make:message            make:auth              make:fixtures
make:test               make:voter             make:subscriber
make:registration-form  make:reset-password    make:state-machine
make:serializer:encoder make:twig-component    make:validator
```

Prefer these over hand-scaffolding — they produce idiomatic 7.4 code.

## Cache + worker management

```
bin/console cache:clear              # rebuild container, configs
bin/console cache:warmup
bin/console messenger:consume async  # start a worker
bin/console messenger:stop-workers   # signal workers to stop (after deploy)
bin/console debug:router             # list routes
bin/console debug:container          # list services
bin/console debug:autowiring         # what can be autowired
bin/console debug:config framework   # current framework config (merged)
bin/console config:dump-reference framework  # reference config skeleton
```

## Gotchas

- After deploy, **always** clear cache and stop workers — long-running
  workers don't pick up new code automatically.
- Don't commit `.env.local`, `var/`, or `vendor/`.
- `composer.json`'s `require` should never include `dev` dependencies — use
  `require-dev`.
- The `App\` namespace is fixed by convention. Don't rename it unless you
  also update `composer.json` and `config/services.yaml`.

## Docs

- Project structure: https://symfony.com/doc/7.4/configuration.html
- Service container: https://symfony.com/doc/7.4/service_container.html
- Console: https://symfony.com/doc/7.4/console.html
