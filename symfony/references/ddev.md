# ddev — local dev environment

## When to use

The project has a `.ddev/config.yaml` file. ddev runs PHP, the database, and
other services inside containers — the host machine's PHP/CLI tools aren't
connected to them.

## Detecting it

```sh
test -f .ddev/config.yaml && echo "ddev project"
```

Check once per session. If present, every shell command in this skill
(and its other references) needs the `ddev` prefix instead of running
directly on the host.

## Command mapping

| Instead of | Run |
|------------|-----|
| `php bin/console …` | `ddev php bin/console …` |
| `composer …` | `ddev composer …` |
| `symfony console …` | `ddev symfony console …` (rarely needed — `ddev php bin/console` covers it) |

`ddev exec <cmd>` and `ddev ssh` also drop you into the web container if a
command doesn't fit the `ddev <tool>` shorthand.

## Why the prefix is non-negotiable for migrations

`ddev php bin/console make:migration` and
`ddev php bin/console doctrine:migrations:migrate` are the two commands
where skipping the `ddev` prefix causes the most damage, not just an error:

- The host PHP process (if one even exists / has the right extensions)
  talks to whatever `DATABASE_URL` resolves to *outside* the container —
  usually nothing, sometimes a stale local DB from a previous project.
- `make:migration` diffs current DB schema against entity mappings. Run
  against the wrong DB, it generates a migration that's wrong for the
  *real* (containerized) database — silently, with no error, because a
  connection succeeded, just to the wrong target.
- `doctrine:migrations:migrate` on the wrong target either fails loudly
  (no connection) or, worse, succeeds against a DB nobody uses, leaving
  the container's actual database un-migrated while the migration table
  says otherwise.

Always confirm `.ddev/config.yaml` exists before running either command,
and prefix with `ddev` when it does. If a command's output doesn't look
right (e.g. `make:migration` reports "No changes detected" when you just
added a field), the first thing to check is whether the prefix was
dropped.

## Other useful commands

| Command | Purpose |
|---------|---------|
| `ddev start` / `ddev stop` | Bring the project's containers up/down |
| `ddev describe` | Show URLs, ports, and container status |
| `ddev logs -f` | Tail web container logs |
| `ddev ssh` | Shell into the web container |
| `ddev mysql` / `ddev psql` | Open a DB CLI session inside the container |
| `ddev launch` | Open the project URL in a browser |

## Mailpit — catching outgoing email

Unlike Redis/RabbitMQ, Mailpit ships built into ddev core — no add-on
needed. It catches every email the project sends instead of delivering it,
with an SMTP listener on port 1025 and a web UI to inspect what was "sent".

| Command | Purpose |
|---------|---------|
| `ddev mailpit` (or `ddev launch -m`) | Open the Mailpit web UI in a browser |

Point `MAILER_DSN` at it for local dev instead of a real transport:

```
# .env.local / .env.dev.local
MAILER_DSN=smtp://localhost:1025
```

From inside the web container `localhost:1025` reaches Mailpit directly —
no project hostname or extra config needed. See [mailer.md](mailer.md) for
the mailer component itself; this is just the local-catch-all transport.

## Add-ons — Redis, RabbitMQ, and others

Services beyond PHP/DB are installed as ddev add-ons
(`ddev add-on get <repository>`), which generate their own containers plus
`ddev`-prefixed commands under `.ddev/commands/`. Check
`.ddev/addon-metadata/` for what's already installed before assuming a
service is missing — `cat .ddev/addon-metadata/<name>/manifest.yaml` shows
the repository and version.

### Redis (`ddev/ddev-redis`)

| Command | Purpose |
|---------|---------|
| `ddev redis-cli [args]` | Run `redis-cli` inside the Redis container, e.g. `ddev redis-cli KEYS '*'` |
| `ddev redis-flush` | `FLUSHALL ASYNC` — clear every key |
| `ddev redis-backend <image> [optimize]` | Swap the backing image (`redis`, `redis-alpine`, `valkey`, `valkey-alpine`, or any Redis-compatible image) |

Corresponds to `symfony/cache` configured with a Redis adapter (see
[cache.md](cache.md)) or Symfony session storage backed by Redis.

### RabbitMQ (`ddev/ddev-rabbitmq`)

| Command | Purpose |
|---------|---------|
| `ddev rabbitmq apply` | Create vhosts/queues/users from `.ddev/rabbitmq/config.yaml` (idempotent — skips what already exists) |
| `ddev rabbitmq wipe` | Delete all vhosts/queues/users except the defaults |
| `ddev rabbitmq watch <target> [interval]` | Live-refresh view of `queues`, `connections`, `consumers`, etc. |
| `ddev rabbitmq launch` | Open the management UI in a browser |
| `ddev rabbitmqctl …` / `ddev rabbitmqadmin …` | Raw CLI passthroughs for anything the wrapper doesn't cover |

Pairs with `symfony/messenger` using the AMQP transport (see
[messenger.md](messenger.md)) — `config.yaml`'s vhosts/queues typically
mirror the transport names in `messenger.yaml`.

**Auto-apply on every `ddev start`** — add a `post-start` hook so the
broker topology is always current without a manual step:

```yaml
# .ddev/config.yaml
hooks:
    post-start:
        - exec-host: ddev rabbitmq apply
```

The same `hooks.post-start` mechanism is the idiomatic way to keep a
Messenger consumer running in the background during development:

```yaml
hooks:
    post-start:
        - exec: symfony run --daemon --watch=config,src,templates,vendor bin/console messenger:consume async -vv
```

### Elasticsearch (`ddev/ddev-elasticsearch`)

No custom `ddev`-prefixed commands — just a container reachable from the
web container at `http://elasticsearch:9200`. Point whatever client you use
(`elasticsearch-php`, FOSElasticaBundle, etc.) at that host/port for local
dev instead of a hosted cluster.

### Gotenberg (`echavaillaz/ddev-gotenberg`)

HTML/Markdown/Office-to-PDF conversion via HTTP — no custom commands
either, just a container at `http://gotenberg:3000`. Typical use: build the
document as an HTML Twig template, then `symfony/http-client` POSTs it to
Gotenberg's `/forms/chromium/convert/html` endpoint to get a PDF back,
instead of shelling out to wkhtmltopdf or a headless-Chrome library
directly (see [http-client.md](http-client.md)).

### No official add-on? Write the docker-compose file by hand

Not every service has a maintained add-on (e.g. an internal observability
stack). Add one anyway by dropping a `.ddev/docker-compose.<name>.yaml`
file — same shape as what `ddev add-on get` generates: a `services:` block
with `com.ddev.site-name` / `com.ddev.approot` labels and `VIRTUAL_HOST`
set to `$DDEV_HOSTNAME` if it needs a browser-reachable URL. `ddev start`
picks up any `docker-compose.*.yaml` in `.ddev/` automatically — no
registration step needed.

To reach *another* local ddev project (e.g. a project split into multiple
repos/services), add an `external_links` entry pointing `ddev-router` at
that project's hostname, rather than hardcoding a port:

```yaml
# .ddev/docker-compose.<name>.yaml
services:
  web:
    external_links:
      - "ddev-router:other-project.ddev.site"
```

## Profiling — xhprof / XHGui

Separate from Symfony's own command profiler (`--profile`, see
[debugging.md](debugging.md)) — this is PHP-level sampling profiling,
built into ddev core, useful for request/page profiling rather than a
single console command.

| Command | Purpose |
|---------|---------|
| `ddev xhprof on` / `off` / `status` / `toggle` | Enable/disable xhprof instrumentation |
| `ddev xhgui` | Start (or check status of) the XHGui UI for browsing collected profiles |

Turn it on, reproduce the slow request, then browse the results in XHGui —
faster than reasoning about a stack trace when the question is "which
function is actually eating the time."

## Docs

- ddev docs: https://ddev.readthedocs.io/
- Symfony + ddev quickstart: https://ddev.readthedocs.io/en/stable/users/quickstart/#symfony
- ddev add-ons overview: https://ddev.readthedocs.io/en/stable/users/extend/additional-services/
- Redis add-on: https://github.com/ddev/ddev-redis
- RabbitMQ add-on: https://github.com/ddev/ddev-rabbitmq
- Elasticsearch add-on: https://github.com/ddev/ddev-elasticsearch
- Gotenberg add-on: https://github.com/echavaillaz/ddev-gotenberg
