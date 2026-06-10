# symfony-skills

A Claude Code skill for Symfony 7.4+. Keeps Claude on idiomatic patterns —
routing, doctrine, security, console, events, testing, framework config —
and reaching for first-party components instead of vendor libraries.

## What's included

- **`SKILL.md`** — always-loaded entry point. Tooling conventions (ddev when
  present), project layout summary, a task index pointing at the right reference
  for each common job, and a catalog of ~40 components grouped by problem area.

- **`references/`** — on-demand reference docs:

  **Task-oriented** (broad Symfony work):
  - `project-layout.md` — directory layout, tooling, autowiring conventions
  - `controllers-and-routing.md` — `#[Route]`, `#[MapRequestPayload]`, `#[MapEntity]`
  - `doctrine.md` — entities, migrations, repositories, query patterns
  - `security.md` — firewalls, authenticators, voters, password hashing
  - `console-commands.md` — `#[AsCommand]`, input/options, `SymfonyStyle`
  - `events.md` — `#[AsEventListener]`, kernel events, custom events
  - `testing.md` — `KernelTestCase`, `WebTestCase`, Panther
  - `framework-config.md` — the full `framework:` config tree

  **Version-gated** (only when the installed Symfony version allows):
  - `whats-new-7.4.md` — curated Symfony 7.4 features (requires >= 7.4)
  - `whats-new-8.1.md` — curated Symfony 8.1 features (requires >= 8.1)

  **Component-oriented** (Symfony component over vendor lib):
  `lock`, `http-client`, `cache`, `mailer`, `messenger`, `notifier`,
  `process`, `filesystem`, `finder`, `serializer`, `validator`,
  `rate-limiter`, `scheduler`, `workflow`, `string`, `uid`, `clock`,
  `html-sanitizer`.

Each reference file follows the same shape: when to use, what you need,
minimal example, common patterns, gotchas, links to the official 7.4 docs.

## Requirements

- Symfony 7.4 or higher. The skill activates when Claude detects a Symfony
  project (composer.json requires `symfony/framework-bundle` at ^7.4, or
  the repo has `bin/console` and `config/packages/`).
- Claude Code with skill support.

## Tooling — ddev when present

If the project has a `.ddev/config.yaml`, Claude will prefix shell commands
with `ddev` (e.g., `ddev php bin/console make:migration`,
`ddev composer require lock`). Without ddev, commands run directly on the
host.

## Installation

Install via [`npx skills`](https://github.com/anthropics/skills):

```sh
npx skills install github:recranet/symfony-skills
```

Or from a local clone:

```sh
git clone https://github.com/recranet/symfony-skills.git
npx skills install ./symfony-skills
```

This copies the skill into your local agent environment. Restart Claude Code
or run `/skill reload` to pick it up.

## Verifying it's active

Open a Symfony 7.4 project in Claude Code and try one of these:

- "Add a controller for `GET /health` that returns `{"ok": true}`" — Claude
  should reach for `#[Route]` and `AbstractController::json()`.
- "I need to dedupe a webhook handler so it doesn't run twice" — Claude
  should propose `symfony/lock` with a TTL, not a Redis SETNX wrapper.
- "Make me a console command that backfills user data" — Claude should
  scaffold an `#[AsCommand]` class using `SymfonyStyle`.
- "Add a Doctrine entity for `Invoice` and a migration" — Claude should run
  `make:entity` followed by `make:migration`, prefixed with `ddev` if
  present.

## Contributing

To add or improve a reference:

1. If a new task/component, add a row to `SKILL.md` (task index or component
   catalog).
2. Create `references/<name>.md` following the existing template (when to use /
   what you need / minimal example / common patterns / gotchas / docs links).
3. Open a PR.

To update existing guidance, edit the reference file directly.

## License

MIT — see [LICENSE](LICENSE).
