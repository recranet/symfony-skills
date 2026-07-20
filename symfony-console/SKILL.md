---
name: symfony-console
description: Console commands with symfony/console - #[AsCommand], arguments and options, SymfonyStyle, exit codes, CommandTester. Use whenever creating or editing a CLI command, bin/console task, cron entrypoint, or backfill script in a Symfony project.
---

# Console commands

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need a CLI command — admin tooling, scheduled jobs, data backfills,
debugging helpers.

## What you need

```
composer require maker --dev  # for make:command
```

Console support is built into `symfony/framework-bundle`.

## Minimal example

```php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(
    name: 'app:rebuild-search-index',
    description: 'Rebuild the search index from scratch',
)]
final class RebuildSearchIndexCommand extends Command
{
    public function __construct(private SearchIndexer $indexer)
    {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);
        $io->title('Rebuilding search index');

        $count = $this->indexer->rebuildAll();

        $io->success("Indexed $count documents.");
        return Command::SUCCESS;
    }
}
```

Run:

```
bin/console app:rebuild-search-index
```

`#[AsCommand]` + autoconfiguration registers the command automatically — no
manual service config.

## Common patterns

### `make:command`

```
bin/console make:command app:rebuild-search-index
```

Generates the class with attributes and a `SymfonyStyle` skeleton.

### Arguments and options

```php
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputOption;

protected function configure(): void
{
    $this
        ->addArgument('email', InputArgument::REQUIRED, 'User email')
        ->addOption('dry-run', null, InputOption::VALUE_NONE, 'Preview only')
        ->addOption('limit', 'l', InputOption::VALUE_REQUIRED, 'Max rows', 1000);
}

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $email  = $input->getArgument('email');
    $dryRun = $input->getOption('dry-run');
    $limit  = (int) $input->getOption('limit');
    // …
}
```

7.4 lets you declare these via the `#[AsCommand]` attribute too:

```php
#[AsCommand(
    name: 'app:invite',
    description: 'Invite a user',
)]
final class InviteCommand extends Command
{
    public function __invoke(
        SymfonyStyle $io,
        #[Argument(description: 'User email')] string $email,
        #[Option(description: 'Skip welcome mail')] bool $skipMail = false,
    ): int {
        // …
        return Command::SUCCESS;
    }
}
```

Prefer the attribute style for new commands — concise and typed.

### `SymfonyStyle` output

```php
$io->title('Big task');
$io->section('Phase 1');
$io->note('FYI');
$io->warning('Mind the gap');
$io->success('Done');
$io->error('Something broke');
$io->table(['Name', 'Count'], $rows);
$io->progressIterate($items, function ($item) { /* … */ });
$io->ask('Email?');
$io->confirm('Continue?', false);
$io->choice('Pick one', ['a', 'b', 'c'], 'a');
```

Always use `SymfonyStyle` — it handles verbose/quiet modes, colour, width,
and CI-friendly output.

### Exit codes

Always return one of:

- `Command::SUCCESS` (0)
- `Command::FAILURE` (1)
- `Command::INVALID` (2) — for bad input

Returning a literal int works but obscures intent.

### Calling another command

Prefer extracting shared logic into a service. If you really need to invoke
another command:

```php
$command = $this->getApplication()->find('cache:clear');
$command->run(new ArrayInput(['--env' => 'prod']), $output);
```

### Lazy loading

`#[AsCommand]` makes commands lazy by default — they're only instantiated
when their name is invoked.

### Long-running commands

For loops, periodically:

```php
$this->getApplication()?->setCatchExceptions(true);

if (\function_exists('pcntl_signal_dispatch')) {
    pcntl_signal_dispatch(); // honour SIGTERM/SIGINT
}
```

For workers, prefer `messenger:consume` (built in) over rolling your own.

## Gotchas

- Command name format: `app:something:something` — `:` namespaces.
- `execute()` and `__invoke()` are alternatives; if you define both,
  `__invoke()` wins.
- Don't echo directly — use `OutputInterface` / `SymfonyStyle`. `echo`
  bypasses verbosity handling.
- Don't `exit(1)` — return `Command::FAILURE` so wrappers and tests see
  the right code.
- For tests, use `Symfony\Bundle\FrameworkBundle\Console\Application` with
  `CommandTester` — see the `symfony-testing` skill.

## Docs

- Console component: https://symfony.com/doc/7.4/console.html
- Lazy commands: https://symfony.com/doc/7.4/console/lazy_commands.html
- Style guide: https://symfony.com/doc/7.4/console/style.html
- Input attributes: https://symfony.com/doc/7.4/console.html#mapping-input-to-command-arguments
