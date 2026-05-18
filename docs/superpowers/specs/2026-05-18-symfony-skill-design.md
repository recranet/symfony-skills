# Symfony Skill — Design

Date: 2026-05-18
Status: Approved, ready for implementation

## Purpose

Provide a Claude Code skill that, when active in a Symfony 7.4+ project, steers
Claude toward Symfony's first-party components instead of vendor-specific
libraries. The skill is distributed as an Anthropic-format skill (`SKILL.md` +
supporting files) installable via `npx skills`.

The motivating example: when a developer needs a lock, Claude should reach for
`symfony/lock` rather than writing a Redis `SETNX` wrapper or pulling in a
third-party lock library. The same applies to HTTP clients (use
`symfony/http-client`, not Guzzle), mail (use `symfony/mailer`, not PHPMailer),
queues (use `symfony/messenger`, not direct AMQP/SQS SDKs), and so on.

## Activation

`SKILL.md` frontmatter `description` field triggers the skill when Claude
detects a Symfony 7.4+ project. Signals:

- `composer.json` requires `symfony/framework-bundle` at ^7.4 or higher
- Repo contains `bin/console`
- Repo contains `config/packages/`

The description explicitly lists the most-replaced libraries (Guzzle,
PHPMailer, direct Redis, etc.) so the skill is selected when Claude is about
to write code in those problem domains.

## Structure

```
symfony-skills/                      # this repo
├── README.md                        # what this repo is, how to install
├── LICENSE
├── .gitignore
├── package.json                     # name, version, repository metadata
├── skills/
│   └── symfony/                     # the installable skill
│       ├── SKILL.md                 # frontmatter + catalog (always loaded)
│       └── references/              # per-component on-demand reference docs
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
    └── 2026-05-18-symfony-skill-design.md   # this file
```

## `SKILL.md` body

Two parts:

### 1. Decision rule (top of file)

A short, imperative section telling Claude that in this project Symfony
components are preferred over vendor-specific equivalents when both fit. Lists
the heuristic: before adding a `composer require` for a third-party library,
check the catalog below — if a Symfony component covers the use case, use it.

### 2. Component catalog (scannable table)

One table grouped by problem area. Each row contains:

| Component | Replaces | Docs | Reference file |
|-----------|----------|------|----------------|

Groups:

- **Network / I/O**: `http-client`, `mailer`, `notifier`, `mime`
- **Data & state**: `cache`, `lock`, `semaphore`, `rate-limiter`,
  `messenger`, `scheduler`, `workflow`
- **Filesystem & process**: `filesystem`, `finder`, `process`
- **Data transformation & validation**: `serializer`, `validator`, `string`,
  `uid`, `html-sanitizer`, `clock`, `expression-language`
- **HTTP / framework essentials**: `http-foundation`, `routing`,
  `security-bundle`, `form`, `twig-bundle`, `translation`, `console`,
  `dependency-injection`, `event-dispatcher`, `yaml`, `config`, `runtime`,
  `asset` / `asset-mapper`, `web-link`
- **Testing / crawling**: `panther`, `dom-crawler`, `browser-kit`,
  `css-selector`

Each docs link points to `https://symfony.com/doc/7.4/...` (pinned to 7.4 so
guidance matches the project's Symfony version).

## Reference files

Each of the 18 reference files in `references/` follows this template:

```
# <Component name>

## When to use
One-liner describing the use case.

## Replaces
Bullet list of common vendor libs / patterns this displaces.

## Install
composer require ...

## Minimal example
<10–30 line code snippet>

## Common patterns
2–4 snippets for the most frequent real-world uses
(e.g. for lock: blocking lock, expiring lock, named store).

## Gotchas
Version notes, traps, what NOT to do.

## Docs
https://symfony.com/doc/7.4/components/<name>.html
```

Reference files are loaded on demand — when Claude is about to write code
that touches the component. They stay out of the always-loaded skill body so
the catalog remains lightweight.

### Reference file set (18 files)

The components most likely to be reached for in vendor-specific form:

1. `lock.md`
2. `http-client.md`
3. `cache.md`
4. `mailer.md`
5. `messenger.md`
6. `notifier.md`
7. `process.md`
8. `filesystem.md`
9. `finder.md`
10. `serializer.md`
11. `validator.md`
12. `rate-limiter.md`
13. `scheduler.md`
14. `workflow.md`
15. `string.md`
16. `uid.md`
17. `clock.md`
18. `html-sanitizer.md`

Catalog-only entries (no reference file): the HTTP/framework essentials and
testing/crawling groups above. These are already idiomatic when working in
Symfony and rarely confused with third-party alternatives.

## Packaging for `npx skills`

`npx skills` installs from a directory (local path or git URL) and copies
each skill folder under `skills/` into `~/.claude/skills/`.

The repo provides:

- `skills/symfony/` as the installable skill folder.
- `package.json` with `name`, `version`, `description`, `repository`, and
  `keywords` so the package is resolvable when referenced as
  `github:recranet/symfony-skills` or installed locally.
- `README.md` documenting:
  - what the skill does and why,
  - prerequisites (Symfony 7.4+),
  - install command: `npx skills install github:recranet/symfony-skills`,
  - alternative: `npx skills install /path/to/clone`,
  - how to contribute (add a reference file, update the catalog).

## Out of scope

- Other Symfony versions (5.x, 6.x). Skill is 7.4-only — `description`
  states this and reference docs link to the 7.4 docs.
- Bundle ecosystem (EasyAdmin, API Platform, etc.). Only first-party Symfony
  components.
- Code-generation guidance (`make:*` recipes). Could be a follow-up skill.
- Upgrade/migration guidance from older Symfony versions. Could be a
  follow-up skill.

## Success criteria

- `npx skills install github:recranet/symfony-skills` places
  `~/.claude/skills/symfony/` with `SKILL.md` and `references/*.md`.
- When Claude is about to write code that calls Guzzle, PHPMailer, a direct
  Redis client, `exec()`, etc., in a Symfony 7.4+ repo, the skill activates
  and Claude proposes the corresponding Symfony component instead.
- For each of the 18 high-misuse-risk components, Claude can load the
  reference file and write idiomatic, working code without further docs
  lookup.
