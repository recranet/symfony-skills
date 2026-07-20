---
name: symfony-process
description: Run external processes with symfony/process - Process and PhpProcess, timeouts, streaming output, error handling. Use instead of exec, shell_exec, or proc_open whenever Symfony code shells out to another program.
---

# symfony/process

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need to run an external command — shell out to `git`, `ffmpeg`,
`imagemagick`, a Python script, a long-running subprocess, etc.

## Replaces

- `exec()`, `shell_exec()`, `system()`, `passthru()`
- `proc_open()` boilerplate
- Backtick operators

## Install

```
composer require symfony/process
```

## Minimal example

```php
use Symfony\Component\Process\Process;
use Symfony\Component\Process\Exception\ProcessFailedException;

$process = new Process(['git', 'rev-parse', 'HEAD']);
$process->run();

if (!$process->isSuccessful()) {
    throw new ProcessFailedException($process);
}

$sha = trim($process->getOutput());
```

## Common patterns

### Pass argv array (NOT a shell string) — prevents injection

```php
// good:
new Process(['ffmpeg', '-i', $userPath, '-vn', $outPath]);

// bad — re-introduces shell injection:
Process::fromShellCommandline("ffmpeg -i $userPath -vn $outPath");
```

Use `fromShellCommandline()` only when you genuinely need shell features
(pipes, redirection) and the inputs are trusted.

### Streaming output

```php
$process = new Process(['rsync', '-av', $src, $dst]);
$process->run(function (string $type, string $buffer) {
    if (Process::ERR === $type) {
        fwrite(STDERR, $buffer);
    } else {
        echo $buffer;
    }
});
```

### Timeouts

```php
$process = new Process(['long-job']);
$process->setTimeout(120);     // seconds; throws if exceeded
$process->setIdleTimeout(10);  // throw if no output for 10s
$process->run();
```

For an unbounded long-running process, `$process->setTimeout(null)`.

### Working directory + env

```php
$process = new Process(
    command: ['npm', 'ci'],
    cwd: $projectDir,
    env: ['NPM_TOKEN' => $token, 'PATH' => getenv('PATH')],
);
$process->mustRun();
```

`mustRun()` is shorthand for `run()` + throw on failure.

### Async / background

```php
$process = new Process(['heavy-job']);
$process->start();
// do other work...
$process->wait();
```

### stdin

```php
$process = new Process(['gpg', '--encrypt', '-r', $recipient]);
$process->setInput($plaintext);
$process->run();
$ciphertext = $process->getOutput();
```

## Gotchas

- Default timeout is 60s. Set explicitly for anything that might run longer.
- `getOutput()` and `getErrorOutput()` include trailing newlines — `trim()`
  if comparing.
- `Process::fromShellCommandline()` is a footgun. Prefer the array form.
- The process inherits the env of the PHP process unless `env` is passed.
  Passing `env` clears it — re-add what you need (e.g., `PATH`).
- On a failed `mustRun()`, the thrown exception's message already includes
  stdout/stderr.

## Docs

https://symfony.com/doc/7.4/components/process.html
