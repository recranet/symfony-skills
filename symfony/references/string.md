# symfony/string

## When to use

You need to manipulate strings safely with Unicode awareness — slugs,
truncation, casing, splitting on graphemes/code points.

## Replaces

- Ad-hoc `mb_*` calls scattered through the code
- Custom slug functions
- `substr`/`str_replace` chains that break on multi-byte input

## Install

```
composer require symfony/string
```

## Minimal example

```php
use function Symfony\Component\String\u;

u('Héllo, World!')->slug();         // 'Hello-World'
u('Café')->upper();                  // 'CAFÉ'
u('foo bar baz')->truncate(7, '…');  // 'foo ba…'
```

`u()` returns a `UnicodeString`; `b()` returns a `ByteString`; `s()` picks
unicode if multibyte is detected.

## Common patterns

### Slug for URLs

```php
use Symfony\Component\String\Slugger\AsciiSlugger;

public function __construct(private AsciiSlugger $slugger) {}

$slug = $this->slugger->slug('Café del Mar')->lower(); // 'cafe-del-mar'
```

The slugger service is autowired. Configure default locale in
`config/packages/translation.yaml` for locale-aware transliteration.

### Truncate

```php
u($text)->truncate(160, '…', UnicodeString::TRUNCATE_WORDS_BEFORE);
```

### Case conversion

```php
u('foo_bar_baz')->camel();        // 'fooBarBaz'
u('FooBarBaz')->snake();          // 'foo_bar_baz'
u('foo-bar')->title(allWords: true); // 'Foo-Bar'
```

### Split / join

```php
u('a, b, c')->split(', ');            // ['a', 'b', 'c'] as UnicodeStrings
u('x')->repeat(3)->join('-');         // 'x-x-x'  (works on array of strings)
```

### Search

```php
u($haystack)->containsAny(['foo', 'bar']);
u($haystack)->startsWith('https://');
u($haystack)->endsWith(['.png', '.jpg']);
```

### Pad / trim

```php
u('5')->padStart(3, '0');   // '005'
u('  x  ')->trim();          // 'x'
```

## Gotchas

- `UnicodeString` operates on grapheme clusters by default — emoji count as
  one "character". If you need code-point or byte semantics, use
  `CodePointString` or `ByteString`.
- All `AbstractString` methods are immutable — they return new instances.
  `$s->trim();` alone does nothing; you need `$s = $s->trim();` or chain.
- `length()` returns grapheme count, not bytes.
- For locale-aware sorting/comparison, see `symfony/intl`.

## Docs

https://symfony.com/doc/7.4/components/string.html
