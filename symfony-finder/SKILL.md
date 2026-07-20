---
name: symfony-finder
description: Find and iterate files with symfony/finder - name/date/size filters, sorting, depth control. Use instead of glob, scandir, or RecursiveIteratorIterator whenever Symfony code walks directories.
---

# symfony/finder

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 or newer — verify the installed version before
> using newer features.

## When to use

You need to find files or directories matching some criteria — by name,
extension, content, depth, date, size.

## Replaces

- `glob()` for anything beyond a simple flat pattern
- `scandir()` + manual recursion
- `RecursiveDirectoryIterator` + `RecursiveIteratorIterator` boilerplate

## Install

```
composer require symfony/finder
```

## Minimal example

```php
use Symfony\Component\Finder\Finder;

$finder = (new Finder())
    ->files()
    ->in($projectDir.'/var/log')
    ->name('*.log')
    ->date('since yesterday');

foreach ($finder as $file) {
    echo $file->getRelativePathname()."\n";
    // $file is a Symfony\Component\Finder\SplFileInfo
}
```

`Finder` is stateful — chain filters, then iterate once.

## Common patterns

### Multiple directories, multiple patterns

```php
$finder = (new Finder())
    ->files()
    ->in([$srcDir, $libDir])
    ->name(['*.php', '*.twig'])
    ->notName('*.test.php');
```

### Depth

```php
$finder->depth('< 3');   // only top 3 levels
$finder->depth(0);        // top level only
```

### Size / date filters

```php
$finder->size('> 1M');
$finder->date('< 30 days ago');
```

### Content match (slow — uses grep semantically)

```php
$finder->contains('TODO');
$finder->notContains('@internal');
```

### Sort

```php
$finder->sortByModifiedTime()->reverseSorting();
```

### Directories only

```php
$dirs = (new Finder())->directories()->in($base)->depth(0);
```

### Empty / count

```php
if (!$finder->hasResults()) { /* … */ }
$n = $finder->count();
```

## Gotchas

- `in()` is required. Forgetting it throws `LogicException`.
- The result is iterable once — iterating again works but re-scans the
  filesystem.
- `name()` uses glob-style by default but accepts regex (`/^foo.+$/`).
- For very large trees, prefer narrowing with `depth()` early.
- `Finder` does not follow symlinks by default — call `followLinks()`.

## Docs

https://symfony.com/doc/7.4/components/finder.html
