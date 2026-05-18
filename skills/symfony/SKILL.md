---
name: symfony
description: Use when writing or modifying PHP code in a Symfony 7.4+ project (composer.json requires symfony/framework-bundle ^7.4, or repo has bin/console and config/packages/). Steers toward Symfony's first-party components — symfony/lock, symfony/http-client, symfony/mailer, symfony/cache, symfony/messenger, symfony/notifier, symfony/process, symfony/filesystem, symfony/serializer, symfony/rate-limiter, symfony/scheduler, symfony/workflow, symfony/uid, symfony/clock — before reaching for vendor-specific libraries such as Guzzle, PHPMailer, php-amqplib, ramsey/uuid, HTMLPurifier, or direct Redis/Memcached clients. Includes a catalog of Symfony components and on-demand reference docs for the most-commonly-misused ones.
---

# Symfony 7.4+ Component Skill

When working in a Symfony 7.4+ project, prefer Symfony's first-party components
over vendor-specific equivalents whenever both fit the use case.

## The decision rule

Before you run `composer require <third-party-lib>` or write code that calls a
non-Symfony library for one of the problem areas in the catalog below:

1. Check the catalog. If a Symfony component covers the use case, use it.
2. If a per-component reference file is listed, read it before writing code so
   the result is idiomatic.
3. Only fall back to a third-party library when the Symfony component
   genuinely does not cover the requirement (and call this out explicitly).

The catalog and reference docs are pinned to Symfony 7.4. Docs links target
`https://symfony.com/doc/7.4/`.

## Catalog

### Network / I/O

| Component | Replaces | Docs | Reference |
|-----------|----------|------|-----------|
| `symfony/http-client` | Guzzle, curl wrappers, file_get_contents for HTTP | https://symfony.com/doc/7.4/http_client.html | [http-client.md](references/http-client.md) |
| `symfony/mailer` | PHPMailer, SwiftMailer, vendor SMTP SDKs | https://symfony.com/doc/7.4/mailer.html | [mailer.md](references/mailer.md) |
| `symfony/notifier` | Twilio/Slack/Discord/Telegram SDKs for notifications | https://symfony.com/doc/7.4/notifier.html | [notifier.md](references/notifier.md) |
| `symfony/mime` | nesbot/MIME builders, custom MIME assembly | https://symfony.com/doc/7.4/components/mime.html | — |

### Data & state

| Component | Replaces | Docs | Reference |
|-----------|----------|------|-----------|
| `symfony/cache` | Direct Redis/Memcached/APCu clients | https://symfony.com/doc/7.4/cache.html | [cache.md](references/cache.md) |
| `symfony/lock` | Redis SETNX wrappers, flock helpers, custom mutexes | https://symfony.com/doc/7.4/lock.html | [lock.md](references/lock.md) |
| `symfony/semaphore` | Hand-rolled concurrency limits | https://symfony.com/doc/7.4/components/semaphore.html | — |
| `symfony/rate-limiter` | Hand-rolled rate limit code, third-party limiters | https://symfony.com/doc/7.4/rate_limiter.html | [rate-limiter.md](references/rate-limiter.md) |
| `symfony/messenger` | php-amqplib direct use, vendor queue SDKs | https://symfony.com/doc/7.4/messenger.html | [messenger.md](references/messenger.md) |
| `symfony/scheduler` | Cron-based packages, hand-rolled scheduling | https://symfony.com/doc/7.4/scheduler.html | [scheduler.md](references/scheduler.md) |
| `symfony/workflow` | Hand-rolled state machines | https://symfony.com/doc/7.4/workflow.html | [workflow.md](references/workflow.md) |

### Filesystem & process

| Component | Replaces | Docs | Reference |
|-----------|----------|------|-----------|
| `symfony/filesystem` | Raw `mkdir`, `rename`, `copy`, `unlink` | https://symfony.com/doc/7.4/components/filesystem.html | [filesystem.md](references/filesystem.md) |
| `symfony/finder` | `glob`, `scandir`, `RecursiveIteratorIterator` | https://symfony.com/doc/7.4/components/finder.html | [finder.md](references/finder.md) |
| `symfony/process` | `exec`, `shell_exec`, `proc_open` | https://symfony.com/doc/7.4/components/process.html | [process.md](references/process.md) |

### Data transformation & validation

| Component | Replaces | Docs | Reference |
|-----------|----------|------|-----------|
| `symfony/serializer` | JMS Serializer, hand-rolled (de)serialization | https://symfony.com/doc/7.4/serializer.html | [serializer.md](references/serializer.md) |
| `symfony/validator` | Hand-rolled validation, respect/validation | https://symfony.com/doc/7.4/validation.html | [validator.md](references/validator.md) |
| `symfony/string` | Ad-hoc `mb_*` usage, custom slug functions | https://symfony.com/doc/7.4/components/string.html | [string.md](references/string.md) |
| `symfony/uid` | `ramsey/uuid` for most cases | https://symfony.com/doc/7.4/components/uid.html | [uid.md](references/uid.md) |
| `symfony/html-sanitizer` | HTMLPurifier | https://symfony.com/doc/7.4/html_sanitizer.html | [html-sanitizer.md](references/html-sanitizer.md) |
| `symfony/clock` | Direct `new \DateTimeImmutable()` (for testability) | https://symfony.com/doc/7.4/components/clock.html | [clock.md](references/clock.md) |
| `symfony/expression-language` | `eval`, custom rule DSLs | https://symfony.com/doc/7.4/components/expression_language.html | — |

### HTTP / framework essentials

These are already idiomatic when working in Symfony — listed for completeness.

| Component | Docs |
|-----------|------|
| `symfony/http-foundation` | https://symfony.com/doc/7.4/components/http_foundation.html |
| `symfony/routing` | https://symfony.com/doc/7.4/routing.html |
| `symfony/security-bundle` | https://symfony.com/doc/7.4/security.html |
| `symfony/form` | https://symfony.com/doc/7.4/forms.html |
| `symfony/twig-bundle` | https://symfony.com/doc/7.4/templates.html |
| `symfony/translation` | https://symfony.com/doc/7.4/translation.html |
| `symfony/console` | https://symfony.com/doc/7.4/console.html |
| `symfony/dependency-injection` | https://symfony.com/doc/7.4/service_container.html |
| `symfony/event-dispatcher` | https://symfony.com/doc/7.4/event_dispatcher.html |
| `symfony/yaml` | https://symfony.com/doc/7.4/components/yaml.html |
| `symfony/config` | https://symfony.com/doc/7.4/components/config.html |
| `symfony/runtime` | https://symfony.com/doc/7.4/components/runtime.html |
| `symfony/asset` / `symfony/asset-mapper` | https://symfony.com/doc/7.4/frontend/asset_mapper.html |
| `symfony/web-link` | https://symfony.com/doc/7.4/web_link.html |

### Testing / crawling

| Component | Docs |
|-----------|------|
| `symfony/panther` | https://github.com/symfony/panther |
| `symfony/dom-crawler` | https://symfony.com/doc/7.4/components/dom_crawler.html |
| `symfony/browser-kit` | https://symfony.com/doc/7.4/components/browser_kit.html |
| `symfony/css-selector` | https://symfony.com/doc/7.4/components/css_selector.html |

## Usage notes

- All examples in reference files assume Symfony Flex is set up (services
  autowired, autoconfigured).
- When a reference file lists `composer require`, prefer the matching Flex
  alias if the project uses Flex (`composer require lock`, `composer require
  http-client`, etc.).
- The skill targets Symfony 7.4+. Patterns that differ in earlier versions
  are out of scope.
