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

## Docs

- ddev docs: https://ddev.readthedocs.io/
- Symfony + ddev quickstart: https://ddev.readthedocs.io/en/stable/users/quickstart/#symfony
