# symfony/html-sanitizer

## When to use

You're accepting HTML from users (rich-text fields, comments, imported
content) and need to strip dangerous markup before storing or rendering.

## Replaces

- `ezyang/htmlpurifier` (HTMLPurifier)
- `strip_tags()` (insufficient — leaves attributes)
- Custom regex-based "sanitisers" (never trust these)

## Install

```
composer require symfony/html-sanitizer
```

## Minimal example

```php
use Symfony\Component\HtmlSanitizer\HtmlSanitizer;
use Symfony\Component\HtmlSanitizer\HtmlSanitizerConfig;

$config = (new HtmlSanitizerConfig())
    ->allowSafeElements()                 // p, br, strong, em, ul, li, a, ...
    ->allowLinkSchemes(['https', 'mailto']);

$sanitizer = new HtmlSanitizer($config);
$safe = $sanitizer->sanitize($untrusted);
```

In a Symfony app, configure once and inject:

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            user_comment:
                allow_safe_elements: true
                allow_link_schemes: ['https', 'mailto']
                allow_relative_links: false
                allow_relative_medias: false
```

```php
use Symfony\Component\HtmlSanitizer\HtmlSanitizerInterface;

public function __construct(private HtmlSanitizerInterface $userCommentSanitizer) {}

$safe = $this->userCommentSanitizer->sanitize($input);
```

Autowire by parameter name `<sanitizerName>Sanitizer`.

## Common patterns

### Whitelist specific elements/attributes

```php
$config = (new HtmlSanitizerConfig())
    ->allowElement('a', ['href', 'title'])
    ->allowElement('img', ['src', 'alt'])
    ->allowElement('p')
    ->allowElement('strong')
    ->blockElement('script')
    ->dropElement('style');
```

`allow` = keep element with listed attributes only.
`block` = drop element but keep children.
`drop` = drop element AND children.

### Restrict link schemes / hosts

```php
$config
    ->allowLinkSchemes(['https'])
    ->allowedLinkHosts(['example.com', 'docs.example.com'])
    ->forceHttpsUrls(true);
```

### Different sanitiser per context

```yaml
framework:
    html_sanitizer:
        sanitizers:
            user_comment:        # restrictive
                allow_safe_elements: true
                allow_link_schemes: ['https', 'mailto']
            editor_content:      # richer
                allow_safe_elements: true
                allow_static_elements: true
                allow_link_schemes: ['https', 'mailto', 'tel']
                allow_relative_links: true
```

### Sanitize as data flows in, render trusted

Sanitise on write, store the safe HTML, render with `|raw` in Twig. Don't
sanitise on read — repeated sanitising can mangle valid input over time.

## Gotchas

- `allowSafeElements()` is a sensible default but explicit whitelists are
  safer for unknown input.
- Sanitising on output instead of on input makes auditing harder and risks
  double-encoding bugs.
- Don't combine `allowSafeElements()` with `blockElement('p')` — order
  matters, but it's clearer to start from nothing and `allowElement()` what
  you want.
- This is HTML sanitisation, not XSS escaping for plain-text contexts.
  For non-HTML contexts use Twig's autoescape.

## Docs

https://symfony.com/doc/7.4/html_sanitizer.html
