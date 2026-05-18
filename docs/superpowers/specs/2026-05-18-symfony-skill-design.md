# Symfony Skill — Design

Date: 2026-05-18
Status: Approved (initial scope), expanded same-day for broader coverage

## Purpose

Provide a Claude Code skill that helps with **any task in a Symfony 7.4+
project** — not only steering toward first-party components, but also
modelling how to write idiomatic Symfony code across controllers, doctrine,
security, forms, console commands, events, testing, and configuration.

The skill is distributed as an Anthropic-format skill (`SKILL.md` +
supporting files) installable via `npx skills`.

Two pillars:

1. **Idiomatic Symfony.** When a task can be done with stock Symfony
   conventions (`#[Route]`, `#[AsCommand]`, autowiring, attributes,
   `make:*` recipes), use them — don't reinvent.
2. **First-party components first.** Where a third-party library is a
   common reach (Guzzle, PHPMailer, php-amqplib, ramsey/uuid,
   HTMLPurifier, direct Redis), prefer the matching Symfony component.

## Activation

`SKILL.md` frontmatter `description` triggers the skill when Claude detects
a Symfony 7.4+ project. Signals:

- `composer.json` requires `symfony/framework-bundle` at ^7.4 or higher
- Repo contains `bin/console`
- Repo contains `config/packages/`

The description names the most-common task surface areas (routing, doctrine,
security, mailer, messenger, console, testing, framework config) and the
most-replaced third-party libraries, so the skill selects on either type of
work.

## Tooling assumption

- If the project has a `.ddev/config.yaml`, prefix shell commands with
  `ddev` (e.g., `ddev php bin/console make:migration`, `ddev composer
  require ...`).
- If ddev is not present, fall back to the host environment. The Symfony
  CLI (`symfony console …`) is acceptable when available; otherwise plain
  `php bin/console …` / `composer …`.

## Structure

```
symfony-skills/
├── README.md
├── LICENSE
├── .gitignore
├── package.json
├── skills/
│   └── symfony/
│       ├── SKILL.md                              # always-loaded
│       └── references/                           # on-demand
│           # Task-oriented references
│           ├── project-layout.md
│           ├── controllers-and-routing.md
│           ├── doctrine.md
│           ├── security.md
│           ├── console-commands.md
│           ├── events.md
│           ├── testing.md
│           ├── framework-config.md
│           # Component-oriented references
│           ├── lock.md
│           ├── http-client.md
│           ├── cache.md
│           ├── mailer.md
│           ├── messenger.md
│           ├── notifier.md
│           ├── process.md
│           ├── filesystem.md
│           ├── finder.md
│           ├── serializer.md
│           ├── validator.md
│           ├── rate-limiter.md
│           ├── scheduler.md
│           ├── workflow.md
│           ├── string.md
│           ├── uid.md
│           ├── clock.md
│           └── html-sanitizer.md
└── docs/superpowers/specs/
    └── 2026-05-18-symfony-skill-design.md
```

## `SKILL.md` body

Four sections:

### 1. Working in a Symfony project

- Project layout (`src/`, `config/`, `templates/`, `migrations/`, `bin/`,
  `public/`, `var/`).
- Tooling: `bin/console`, Flex, env vars, `composer` aliases, ddev wrapping
  when present.
- Convention defaults: autowire on, autoconfigure on, services private,
  PSR-4 `App\` autoload.

### 2. Common tasks

A scannable list with one-line "use X" guidance and a link to either a
reference file or the canonical 7.4 docs:

- Add a route / controller → `controllers-and-routing.md`
- Add an entity / migration → `doctrine.md`
- Add a console command → `console-commands.md`
- React to an event → `events.md`
- Add validation → `references/validator.md`
- Send email → `references/mailer.md`
- Run background work → `references/messenger.md`
- Schedule recurring work → `references/scheduler.md`
- Secure an endpoint → `security.md`
- Write a test → `testing.md`
- Configure the framework → `framework-config.md`
- … (one row per common task)

### 3. Component catalog

The existing catalog (unchanged). Grouped tables: Network/IO, Data & state,
Filesystem & process, Data transformation & validation, HTTP/framework
essentials, Testing/crawling. Each row: component → replaces → docs link →
reference file (if any).

### 4. Decision rule

A short imperative section telling Claude:

1. For task-shaped work, follow the matching task reference.
2. Before reaching for a third-party library, check the component catalog.
3. Pin all examples to Symfony 7.4 docs.

## Reference files

All reference files use the same template:

```
# <Title>

## When to use
## What you need
## Minimal example
## Common patterns
## Gotchas
## Docs
```

**Task references** focus on the canonical Symfony workflow for the task
(attributes, services, console commands, conventions).

**Component references** focus on which vendor lib they replace and how to
write idiomatic code with the component.

## Packaging for `npx skills`

`npx skills` installs from a directory (local path or git URL) and copies
each skill folder under `skills/` into `~/.claude/skills/`. Repo provides:

- `skills/symfony/` as the installable skill folder.
- `package.json` with name, version, description, repository, keywords.
- `README.md` documenting install commands and verification.

## Out of scope

- Symfony 5.x / 6.x (skill is 7.4+ only).
- Third-party bundle ecosystem (EasyAdmin, API Platform, Sylius, etc.).
- Code-generation guidance for `make:*` beyond the recipes mentioned in
  task references — the `MakerBundle` is the source of truth.
- Frontend frameworks consumed via AssetMapper (React, Vue stacks).

## Success criteria

- `npx skills install github:recranet/symfony-skills` places
  `~/.claude/skills/symfony/` with `SKILL.md` and 26 reference files.
- For broad Symfony tasks ("add a route", "add a console command", "secure
  this endpoint", "write a test for this controller") Claude can load the
  matching task reference and produce idiomatic 7.4 code.
- For component-replacement scenarios (Guzzle, PHPMailer, direct Redis,
  etc.) Claude proposes the Symfony component instead.
- When ddev is present in the project, Claude prefixes shell commands with
  `ddev`.
