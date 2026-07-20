# symfony-skills

Claude Code skills for Symfony 7.4, 8.0 or newer. Keeps Claude on idiomatic patterns —
routing, doctrine, security, console, events, testing, framework config —
and reaching for first-party components instead of vendor libraries.

## What's included

This repo is a **skill family**: a core `symfony/` skill for cross-cutting
concerns, plus one `symfony-<component>/` skill per Symfony component. Each
directory is a named skill (`SKILL.md`, optionally with `references/`), so
`npx skills` can bundle them individually or all together.

### Core skill

- **`symfony/`** — the hub. Tooling conventions (ddev when present), version
  gating, project layout, a task index routing every common job to the right
  reference or companion skill, and a catalog of ~40 components grouped by
  problem area. On-demand references:
  - `project-layout.md` — directory layout, tooling, autowiring conventions
  - `controllers-and-routing.md` — `#[Route]`, `#[MapRequestPayload]`, `#[MapEntity]`
  - `events.md` — `#[AsEventListener]`, kernel events, custom events
  - `framework-config.md` — the full `framework:` config tree
  - `debugging.md` — debug commands, profiling
  - `ddev.md` — running everything through ddev, Mailpit, xhprof/XHGui
  - `whats-new-7.4.md` / `whats-new-8.1.md` — version-gated features

### Component skills

One skill per component, grouped by Symfony component. Each contains the
full when-to-use / install / minimal example / patterns / gotchas guidance:

| Skill | Covers |
|-------|--------|
| `symfony-doctrine` | Entities, repositories, migrations, query patterns; `references/imports.md` for memory-safe batch imports |
| `symfony-console` | `#[AsCommand]`, input/options, `SymfonyStyle` |
| `symfony-security` | Firewalls, authenticators, voters, password hashing |
| `symfony-testing` | `KernelTestCase`, `WebTestCase`, Panther |
| `symfony-messenger` | Messages, handlers, transports, routing conventions, flush ownership |
| `symfony-scheduler` | `#[AsPeriodicTask]` / `#[AsCronTask]`, schedules |
| `symfony-http-client` | Outbound HTTP, retries, scoped clients, `MockHttpClient` |
| `symfony-mailer` | Email, transports, Mailpit in dev |
| `symfony-notifier` | Chat / SMS / push notifications |
| `symfony-cache` | Pools, cache contracts, Redis/Memcached/APCu |
| `symfony-lock` | Mutexes, TTLs, blocking acquire |
| `symfony-rate-limiter` | Token bucket, windows, reservations |
| `symfony-workflow` | State machines, transitions, guards |
| `symfony-serializer` | Normalizers, groups, DTO mapping |
| `symfony-validator` | Constraint attributes, custom constraints |
| `symfony-string` | `UnicodeString`, slugger |
| `symfony-uid` | UUIDs / ULIDs, Doctrine types |
| `symfony-clock` | Testable time, `MockClock` |
| `symfony-html-sanitizer` | Sanitizing user-submitted HTML |
| `symfony-process` | Running external processes |
| `symfony-finder` | Finding / iterating files |
| `symfony-filesystem` | Filesystem operations, atomic writes |

Each skill follows the same shape: when to use, what you need, minimal
example, common patterns, gotchas, links to the official Symfony docs.

## Requirements

- Symfony 7.4 or higher. The skills activate when Claude detects a Symfony
  project (composer.json requires `symfony/framework-bundle` at ^7.4, or
  the repo has `bin/console` and `config/packages/`) and the task matches
  a skill's domain.
- Claude Code with skill support.

## Tooling — ddev when present

If the project has a `.ddev/config.yaml`, Claude will prefix shell commands
with `ddev` (e.g., `ddev php bin/console make:migration`,
`ddev composer require lock`). Without ddev, commands run directly on the
host. Every component skill carries this convention; the full mapping lives
in `symfony/references/ddev.md`.

## Installation

Install via [`npx skills`](https://github.com/anthropics/skills):

```sh
npx skills add recranet/symfony-skills
```

Or from a local clone:

```sh
git clone https://github.com/recranet/symfony-skills.git
npx skills add ./symfony-skills
```

This copies the skills into your local agent environment. Install everything
for full coverage, or cherry-pick the component skills you need — the core
`symfony/` skill is recommended in all cases since it routes tasks to the
others. Restart Claude Code or run `/skill reload` to pick them up.

## Verifying it's active

Open a Symfony 7.4 or 8.0 project in Claude Code and try one of these:

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

To add or improve guidance:

1. For a new component, create `symfony-<component>/SKILL.md` following the
   existing template (frontmatter with a triggering-oriented description,
   then when to use / what you need / minimal example / common patterns /
   gotchas / docs links), and add it to the task index and component
   catalog in `symfony/SKILL.md`.
2. For cross-cutting guidance, edit or add a reference under
   `symfony/references/`.
3. Open a PR.

To update existing guidance, edit the skill or reference file directly.

## License

MIT — see [LICENSE](LICENSE).
