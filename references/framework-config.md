# Framework configuration

## When to use

You need to configure the `framework:` tree in `config/packages/*.yaml` —
session, mailer, messenger, cache, http_client, validation, serializer,
csrf, profiler, assets, lock, rate_limiter, scheduler, etc.

Full canonical reference (always pin to this when in doubt):

**https://symfony.com/doc/current/reference/configuration/framework.html**

For 7.4-specific defaults: https://symfony.com/doc/7.4/configuration.html

## Where it lives

```
config/
└── packages/
    ├── framework.yaml          # main config
    ├── mailer.yaml             # framework.mailer.*
    ├── messenger.yaml          # framework.messenger.*
    ├── cache.yaml              # framework.cache.*
    ├── http_client.yaml        # framework.http_client.*
    ├── lock.yaml               # framework.lock.*
    ├── rate_limiter.yaml       # framework.rate_limiter.*
    ├── scheduler.yaml          # framework.scheduler.*
    └── routing.yaml            # framework.router.*
```

Per-environment overrides go in `config/packages/<env>/*.yaml` (e.g.,
`config/packages/test/framework.yaml`).

## How to read the current and reference config

```sh
bin/console debug:config framework                      # merged active config
bin/console debug:config framework messenger            # zoom in
bin/console config:dump-reference framework             # full reference (defaults)
bin/console config:dump-reference framework http_client # subtree only
```

`config:dump-reference` is authoritative — it prints exactly what the
installed version accepts.

## Top-level keys (the framework: tree)

Each key below maps to a section of the framework reference. Use this as a
"what's available" index — for any non-trivial config, run
`config:dump-reference framework <key>` and read the official docs page.

| Key | Configures | Docs anchor |
|-----|------------|-------------|
| `secret` | App-wide secret (CSRF, remember-me, …) | `#secret` |
| `default_locale` | Default locale | `#default-locale` |
| `enabled_locales` | Locale whitelist | `#enabled-locales` |
| `set_locale_from_accept_language` | Auto-detect locale from header | `#set-locale-from-accept-language` |
| `set_content_language_from_locale` | Set `Content-Language` header | `#set-content-language-from-locale` |
| `http_method_override` | Honour `_method` in forms | `#http-method-override` |
| `trust_x_sendfile_type_header` | nginx X-Sendfile | `#trust-x-sendfile-type-header` |
| `trusted_proxies` / `trusted_headers` | Behind a reverse proxy | `#trusted-proxies` |
| `trusted_hosts` | Allowed Host headers | `#trusted-hosts` |
| `handle_all_throwables` | Wrap Throwable not just Exception | `#handle-all-throwables` |
| `error_controller` | Custom error controller | `#error-controller` |
| `ide` | Open file URLs in IDE | `#ide` |
| `test` | Enable test client | `#test` |
| `session` | Sessions: storage, cookie, gc | `#session` |
| `assets` | Versioning, base path/url | `#assets` |
| `asset_mapper` | AssetMapper paths, public_prefix | `#asset-mapper` |
| `translator` | Translation paths, fallbacks | `#translator` |
| `validation` | Validator config | `#validation` |
| `annotations` | (Removed in 7.4 — use attributes) | — |
| `serializer` | Default groups, name converter | `#serializer` |
| `property_access` | Magic methods, throw_exception | `#property-access` |
| `property_info` | Property info extractor wiring | `#property-info` |
| `type_info` | Type info component | `#type-info` |
| `cache` | Pools and adapters | `#cache` |
| `router` | Strict requirements, default URI | `#router` |
| `request` | Custom request formats | `#request` |
| `form` | CSRF, legacy error messages | `#form` |
| `csrf_protection` | CSRF on/off | `#csrf-protection` |
| `esi`, `ssi`, `fragments` | Edge-side includes, fragments | `#esi` |
| `profiler` | Web profiler | `#profiler` |
| `routing` | (See `router`) | `#routing` |
| `mailer` | DSNs, envelope, headers | `#mailer` |
| `notifier` | Channels, transports, recipients | `#notifier` |
| `messenger` | Transports, routing, buses, retry | `#messenger` |
| `scheduler` | Schedulers config | `#scheduler` |
| `workflows` | State machines and workflows | `#workflows` |
| `lock` | Lock stores | `#lock` |
| `semaphore` | Semaphore stores | `#semaphore` |
| `rate_limiter` | Limiter policies, storage | `#rate-limiter` |
| `http_client` | Default + scoped clients | `#http-client` |
| `webhook` | Inbound webhook receiver | `#webhook` |
| `remote-event` | Remote event consumers | `#remote-event` |
| `web_link` | HTTP/2 Push headers | `#web-link` |
| `html_sanitizer` | Per-name sanitiser configs | `#html-sanitizer` |
| `uid` | Default factory (UUID v1/4/6/7), Ulid | `#uid` |
| `php_errors` | log_channel, error logging level | `#php-errors` |
| `disallow_search_engine_index` | Send `X-Robots-Tag: noindex` | `#disallow-search-engine-index` |
| `exceptions` | Exception → status code mapping | `#exceptions` |

Each link is anchored relative to
https://symfony.com/doc/current/reference/configuration/framework.html.

## Common excerpts

### Sessions

```yaml
framework:
    session:
        handler_id: null            # native PHP sessions
        cookie_secure: auto
        cookie_samesite: lax
        cookie_lifetime: 0          # browser session
        gc_maxlifetime: 86400
        save_path: '%kernel.project_dir%/var/sessions/%kernel.environment%'
```

For Redis-backed sessions:

```yaml
framework:
    session:
        handler_id: '%env(REDIS_URL)%'
```

### Behind a reverse proxy

```yaml
framework:
    trusted_proxies: '%env(TRUSTED_PROXIES)%'   # 127.0.0.1, 10.0.0.0/8, …
    trusted_headers: ['x-forwarded-for', 'x-forwarded-host', 'x-forwarded-proto', 'x-forwarded-port', 'x-forwarded-prefix']
```

### Exceptions → HTTP status

```yaml
framework:
    exceptions:
        App\Exception\InvariantViolated:
            log_level: warning
            status_code: 422
```

### Uid default factory

```yaml
framework:
    uid:
        default_uuid_version: 7
        time_based_uuid_version: 7
        name_based_uuid_version: 5
        name_based_uuid_namespace: '%env(UUID_NS)%'
```

### Web profiler (dev only)

```yaml
# config/packages/dev/web_profiler.yaml
when@dev:
    web_profiler:
        toolbar: true
        intercept_redirects: false
    framework:
        profiler:
            only_exceptions: false
            collect_serializer_data: true
```

## Environment-conditional config (`when@`)

7.4 syntax for env-specific blocks within a single file:

```yaml
framework:
    cache:
        app: cache.adapter.filesystem

when@prod:
    framework:
        cache:
            app: cache.adapter.redis
            default_redis_provider: '%env(REDIS_URL)%'

when@test:
    framework:
        cache:
            app: cache.adapter.array
```

Prefer this over scattering across `config/packages/<env>/`.

## Gotchas

- `framework.annotations` is gone — use attributes.
- `framework.session.cookie_samesite: lax` is the safe default. `strict`
  breaks OAuth callback flows.
- `trusted_proxies` is required when running behind a reverse proxy, or
  `Request::isSecure()` and client IP detection are wrong.
- Editing config requires `cache:clear` if not in dev.
- For unknown keys, `config:dump-reference framework <key>` is faster
  than guessing.

## Docs

- **Framework reference (canonical):**
  https://symfony.com/doc/current/reference/configuration/framework.html
- 7.4 configuration overview: https://symfony.com/doc/7.4/configuration.html
- Environment variables: https://symfony.com/doc/7.4/configuration/env_var_processors.html
- `when@<env>`: https://symfony.com/doc/7.4/configuration.html#configuration-environments
