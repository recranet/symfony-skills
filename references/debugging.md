# Debugging — inspect runtime state

## When to use

Something doesn't behave as expected — a route isn't resolving, a service
isn't autowired, a config value isn't what you assumed, the database
contains a row you didn't expect, or a request is failing in a way the
exception page doesn't explain.

Use the first-party tools below before reaching for `dd()` everywhere or
tailing logs manually.

## Tooling — ddev when present

If the project has `.ddev/config.yaml`, prefix every command in this file
with `ddev exec` (so `php bin/console ...` becomes
`ddev exec php bin/console ...`). Otherwise run on the host directly.

## Inspecting the database — `dbal:run-sql`

For local development inspection only. Read-only queries against the
dev database:

```sh
ddev exec php bin/console dbal:run-sql 'SELECT id, email, created_at FROM "user" ORDER BY created_at DESC LIMIT 20'
ddev exec php bin/console dbal:run-sql "SELECT COUNT(*) FROM orders WHERE status = 'pending'"
ddev exec php bin/console dbal:run-sql 'SHOW TABLES'                    # MySQL
ddev exec php bin/console dbal:run-sql "\d+ user"                       # not supported — use information_schema instead
ddev exec php bin/console dbal:run-sql "SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'user'"
```

Useful flags:

- `--connection=<name>` — pick a specific DBAL connection if more than one
  is configured.
- `--force-fetch` — force fetching results (DDL/DML statements normally
  only report affected rows).

### NEVER use `dbal:run-sql` for schema changes

`dbal:run-sql` is for **inspection only**. Do not use it to alter schema,
backfill production data, or "just quickly fix" a column type. All schema
changes go through migrations:

```sh
# After editing an entity:
ddev exec php bin/console make:migration

# Review the generated file under migrations/, then:
ddev exec php bin/console doctrine:migrations:migrate
```

Running raw `ALTER TABLE` / `CREATE TABLE` via `dbal:run-sql` produces
drift between the schema and the migration history — the next
`make:migration` will generate a confusing diff, and other environments
will be out of sync. If you're tempted to do it, write the migration
instead.

## `debug:*` family

`bin/console list debug` shows everything. The ones you'll reach for most:

| Command | Answers |
|---------|---------|
| `debug:router` | Which route handles `GET /foo/bar`? What's the controller? |
| `debug:router <name>` | Full details for one route — path, methods, requirements, defaults. |
| `debug:router --show-controllers` | Route → controller mapping at a glance. |
| `router:match /foo/bar` | What route (if any) would Symfony pick for this path? |
| `debug:container` | List every service ID — grep for the one you want. |
| `debug:container App\Service\Foo` | Show class, tags, args for one service. |
| `debug:container --tag=kernel.event_listener` | All services with a given tag. |
| `debug:container --parameters` | All container parameters. |
| `debug:autowiring` | What types can I type-hint to autowire? |
| `debug:autowiring Logger` | Filter to matches containing "Logger". |
| `debug:config` | Resolved configuration tree (all bundles). |
| `debug:config framework` | One bundle's resolved config. |
| `debug:config framework session` | One subtree. |
| `config:dump-reference framework` | Reference config (with every option, even defaults), not the resolved one. |
| `debug:event-dispatcher` | All listeners for all events, in priority order. |
| `debug:event-dispatcher kernel.request` | Listeners for one event. |
| `debug:firewall <name>` | Firewall config + listeners, for security debugging. |
| `debug:twig` | Twig globals, filters, functions, tests available in templates. |
| `debug:translation en` | Missing / unused translation keys for a locale. |
| `debug:dotenv` | Which `.env*` files are loaded and what each variable resolves to. |
| `debug:messenger` | Configured Messenger transports + their routing. |
| `debug:scheduler` | Configured schedules and their next run. |
| `debug:validator 'App\Entity\Foo'` | Constraints declared on an entity. |
| `debug:form` | Form types available + their options. |
| `secrets:list --reveal` | Decrypted secrets (env-scoped vault). |

Anything `debug:*` is read-only — safe to run anywhere.

## Doctrine-specific debugging

| Command | Use |
|---------|-----|
| `doctrine:schema:validate` | Are the mapping metadata and DB schema in sync? Run before assuming a migration is needed. |
| `doctrine:mapping:info` | List every mapped entity Doctrine sees. |
| `doctrine:query:dql 'SELECT p FROM App\Entity\Product p WHERE p.priceCents < 1000'` | Run DQL ad hoc. |
| `doctrine:migrations:status` | Which migrations have run, which are pending. |
| `doctrine:migrations:list` | All migrations + their executed/not-executed state. |
| `doctrine:cache:clear-metadata` / `:clear-query` / `:clear-result` | Clear ORM caches when stale entries are masking changes. |

For complex queries against the dev DB, prefer `doctrine:query:dql` over
`dbal:run-sql` — it goes through hydration so you see what the
application would see.

## VarDumper — `dump()` and `dd()`

Use inside controllers, services, Twig templates, or `bin/console` commands
during development. **Do not commit `dd()` calls** — they halt execution
and will tank requests.

```php
dump($value);   // dumps and continues
dd($value);     // dumps and dies
```

In Twig:

```twig
{{ dump(user) }}
{{ dump() }}    {# dump all variables in scope #}
```

In a `bin/console` command, dumps go to stdout. Over HTTP, output them to
the response or remove before committing.

## Logs

```sh
# Tail dev log:
ddev exec tail -f var/log/dev.log

# Search for errors over the last run:
ddev exec grep -E '(ERROR|CRITICAL)' var/log/dev.log
```

Channels are configured in `config/packages/monolog.yaml`. The `event`,
`doctrine`, `security`, `request`, and `cache` channels are useful when
narrowing down what produced a line.

## Server / runtime

| Command | Use |
|---------|-----|
| `cache:clear` | After config / annotation / route changes if the app behaves stale. |
| `cache:warmup` | Pre-warm caches; useful after `cache:clear` in CI. |
| `about` | Symfony version, PHP version, env, debug mode, cache dir, log dir, key bundles. |
| `lint:yaml config/` | Validate YAML before deploy. |
| `lint:twig templates/` | Validate Twig syntax. |
| `lint:container` | Verify every service's args resolve at compile time. |

## Gotchas

- `dbal:run-sql` is **dev inspection only**. Schema changes go through
  `make:migration` → `doctrine:migrations:migrate`. Never alter the schema
  via raw SQL — it desyncs migration history across environments.
- Quote SQL carefully under the shell. PostgreSQL reserved-word table
  names like `user` need double quotes; the shell needs single quotes
  wrapping the whole statement so the double quotes survive.
- `debug:config` shows the **resolved** config (after env-var
  substitution, defaults, overrides). `config:dump-reference` shows the
  **reference** schema. They answer different questions.
- After editing `services.yaml` or attributes used for service wiring,
  `cache:clear` if the container appears stale — autoload + dev cache
  usually handle it, but not always after large refactors.
- `dump()`/`dd()` left in committed code will surface in code review at
  best and break tests/requests at worst. Strip them before pushing.

## Docs

- VarDumper: https://symfony.com/doc/7.4/components/var_dumper.html
- `debug:*` commands: https://symfony.com/doc/7.4/console.html
- Doctrine migrations: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
- DBAL `dbal:run-sql`: https://www.doctrine-project.org/projects/doctrine-dbal/en/current/reference/configuration.html
