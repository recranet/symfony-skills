# symfony-skills

A Claude Code skill that steers toward Symfony's first-party components when
working in a Symfony 7.4+ project. Instead of reaching for Guzzle, PHPMailer,
direct Redis clients, hand-rolled state machines, or `ramsey/uuid`, Claude
will prefer `symfony/http-client`, `symfony/mailer`, `symfony/cache`,
`symfony/workflow`, `symfony/uid`, and friends.

## What's included

- **`skills/symfony/SKILL.md`** — catalog of ~40 Symfony components grouped by
  problem area (network/IO, data & state, filesystem & process, validation,
  testing, …). Each entry lists what the component replaces and links to the
  official docs.
- **`skills/symfony/references/`** — on-demand reference docs for the 18
  components most commonly reached for in vendor-specific form:
  `lock`, `http-client`, `cache`, `mailer`, `messenger`, `notifier`,
  `process`, `filesystem`, `finder`, `serializer`, `validator`,
  `rate-limiter`, `scheduler`, `workflow`, `string`, `uid`, `clock`,
  `html-sanitizer`.

Each reference doc follows the same structure: when to use, what it replaces,
install command, minimal example, common patterns, gotchas, link to the 7.4
documentation.

## Requirements

- Symfony 7.4 or higher. The skill activates when Claude detects a Symfony
  project (composer.json requires `symfony/framework-bundle` at ^7.4, or the
  repo has `bin/console` and `config/packages/`).
- Claude Code with skill support.

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

This copies `skills/symfony/` to `~/.claude/skills/symfony/`. Restart your
Claude Code session (or `/skill reload`) to pick it up.

## Verifying it's active

Open a Symfony 7.4 project in Claude Code and ask:

> "I need to dedupe a webhook handler so it doesn't run twice. What's the
> right approach here?"

Claude should propose `symfony/lock` (with a TTL'd lock acquired via
`LockFactory`) rather than a Redis SETNX wrapper or a custom mutex.

## Contributing

To add a new component reference:

1. Add a row to the appropriate table in `skills/symfony/SKILL.md`.
2. Create `skills/symfony/references/<component>.md` following the template
   used by the existing files (when to use / replaces / install / minimal
   example / common patterns / gotchas / docs link).
3. Open a PR.

To update the catalog without adding a new reference, edit the table in
`SKILL.md` directly.

## License

MIT — see [LICENSE](LICENSE).
