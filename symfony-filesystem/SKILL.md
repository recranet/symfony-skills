---
name: symfony-filesystem
description: Filesystem operations with symfony/filesystem - mkdir, copy, atomic writes with dumpFile, tempnam. Use instead of raw mkdir/rename/copy/unlink/file_put_contents whenever Symfony code touches the filesystem.
---

# symfony/filesystem

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need to manipulate files and directories — create, copy, move, delete,
chmod, symlink. Cross-platform with sensible errors.

## Replaces

- Raw `mkdir`, `rename`, `copy`, `unlink`, `chmod`, `symlink`
- Recursive-delete helpers
- Custom path-joining string concatenation

## Install

```
composer require symfony/filesystem
```

## Minimal example

```php
use Symfony\Component\Filesystem\Filesystem;
use Symfony\Component\Filesystem\Path;

public function __construct(private Filesystem $fs) {}

$this->fs->mkdir(Path::join($baseDir, 'reports', '2026'));
$this->fs->dumpFile(Path::join($baseDir, 'reports', '2026', 'q1.csv'), $csv);
```

`Filesystem` and `Path` are autowired.

## Common patterns

### Atomic write

`dumpFile()` writes to a temp file then renames — readers never see a
half-written file.

```php
$this->fs->dumpFile($path, $contents);
```

### Mirror / sync a directory

```php
$this->fs->mirror($src, $dst, options: ['override' => true, 'delete' => true]);
```

### Remove recursively

```php
$this->fs->remove($pathOrIterable);
```

Accepts a path, array, or `Finder` iterator.

### Idempotent mkdir

```php
$this->fs->mkdir($path); // safe if already exists
```

### Symlinks

```php
$this->fs->symlink($target, $linkPath, copyOnWindows: true);
$this->fs->readlink($linkPath, canonicalize: true);
```

### Path helpers

```php
Path::join('a', 'b', 'c');                    // 'a/b/c'
Path::makeRelative('/var/app/log', '/var/app'); // 'log'
Path::canonicalize('/a/./b/../c');             // '/a/c'
Path::isAbsolute('/x');                        // true
```

### Temp file/dir

```php
$tmp = $this->fs->tempnam(sys_get_temp_dir(), 'export_', '.csv');
```

## Gotchas

- All methods throw `IOExceptionInterface` on failure — don't check return
  values, catch the exception or let it bubble.
- `remove()` on a non-existent path is a no-op (no exception).
- `dumpFile()` requires the directory to be writable; it creates intermediate
  directories itself.
- `Path` is a separate class — don't reinvent path-joining with
  `DIRECTORY_SEPARATOR` concatenation.

## Docs

https://symfony.com/doc/7.4/components/filesystem.html
