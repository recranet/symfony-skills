---
name: symfony-http-client
description: Outbound HTTP with symfony/http-client - requests, JSON handling, retries, scoped clients, streaming, MockHttpClient in tests. Use whenever a Symfony project calls an external API or fetches a URL, even if the user asks for Guzzle, curl, or file_get_contents.
---

# symfony/http-client

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need to make outbound HTTP calls — fetch JSON from an API, post a webhook,
download a file, stream a response.

## Replaces

- Guzzle (`guzzlehttp/guzzle`)
- Direct `curl_*` usage
- `file_get_contents($url, …)` for HTTP
- Vendor SDKs that only wrap HTTP (consider replacing if the SDK is thin)

## Install

```
composer require symfony/http-client
```

## Minimal example

```php
use Symfony\Contracts\HttpClient\HttpClientInterface;

public function __construct(private HttpClientInterface $client) {}

public function fetchUser(int $id): array
{
    $response = $this->client->request('GET', "https://api.example.com/users/$id");
    return $response->toArray(); // throws on non-2xx
}
```

`HttpClientInterface` is autowired to a shared client.

## Common patterns

### Scoped client (per-API base URI, auth, default headers)

Configure once, inject by parameter name:

```yaml
# config/packages/http_client.yaml
framework:
    http_client:
        scoped_clients:
            github.client:
                base_uri: 'https://api.github.com/'
                auth_bearer: '%env(GITHUB_TOKEN)%'
                headers:
                    Accept: 'application/vnd.github+json'
```

```php
public function __construct(private HttpClientInterface $githubClient) {}

$this->githubClient->request('GET', 'repos/symfony/symfony');
```

### Retry on transient failure

```yaml
framework:
    http_client:
        scoped_clients:
            flaky.client:
                base_uri: 'https://flaky.example.com/'
                retry_failed:
                    max_retries: 3
                    delay: 500
                    multiplier: 2
```

### JSON body + typed response

```php
$response = $this->client->request('POST', $url, [
    'json' => ['name' => 'Alice'],
]);

if (201 !== $response->getStatusCode()) {
    throw new \RuntimeException('Unexpected status');
}

return $response->toArray();
```

### Streaming a download

```php
$response = $this->client->request('GET', $url);
$fh = fopen($path, 'w');
foreach ($this->client->stream($response) as $chunk) {
    fwrite($fh, $chunk->getContent());
}
fclose($fh);
```

### Concurrency (async by default)

```php
$responses = [];
foreach ($urls as $url) {
    $responses[] = $this->client->request('GET', $url);
}
foreach ($this->client->stream($responses) as $response => $chunk) {
    if ($chunk->isLast()) {
        $results[] = $response->toArray();
    }
}
```

## Gotchas

- All `$response` methods (`getStatusCode`, `getContent`, `toArray`) are
  blocking — they wait for headers/body. Requests themselves are non-blocking,
  so fire many then drain via `stream()`.
- `toArray()` and `getContent()` throw on non-2xx by default. Pass `false` to
  suppress, or catch `ClientExceptionInterface` / `ServerExceptionInterface`.
- Don't reuse the cURL handle layer manually — let the HttpClient pool it.
- For tests, use `Symfony\Component\HttpClient\MockHttpClient` rather than
  mocking the interface.

## Docs

https://symfony.com/doc/7.4/http_client.html
