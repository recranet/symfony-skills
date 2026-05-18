# Controllers & Routing

## When to use

You need to expose an HTTP endpoint — receive a request, return a response.

## What you need

```
composer require symfony/framework-bundle  # already there in any Symfony app
```

Use attribute-based routing exclusively in 7.4+. YAML/XML routing is legacy.

## Minimal example

```php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class HealthController extends AbstractController
{
    #[Route('/health', methods: ['GET'])]
    public function __invoke(): JsonResponse
    {
        return new JsonResponse(['ok' => true]);
    }
}
```

## Common patterns

### Route parameters + requirements

```php
#[Route('/users/{id<\d+>}', methods: ['GET'])]
public function show(int $id): Response
{
    // …
}
```

Inline regex with `<>` is the 7.4 way; avoid the older `requirements: [...]`
array unless you have many.

### Auto-converting parameters to entities

```php
use Symfony\Bridge\Doctrine\Attribute\MapEntity;

#[Route('/users/{id}', methods: ['GET'])]
public function show(#[MapEntity] User $user): Response
{
    return $this->json($user);
}
```

Or by a different property:

```php
#[Route('/users/{slug}', methods: ['GET'])]
public function show(#[MapEntity(mapping: ['slug' => 'slug'])] User $user): Response
```

### Request body → DTO, validated

```php
use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;

#[Route('/users', methods: ['POST'])]
public function create(#[MapRequestPayload] CreateUserDto $dto): Response
{
    // $dto is deserialized + validated; on failure 422 is returned
}
```

For query strings:

```php
use Symfony\Component\HttpKernel\Attribute\MapQueryString;

#[Route('/search', methods: ['GET'])]
public function search(#[MapQueryString] SearchFilters $filters): Response
```

### Returning JSON

```php
return $this->json(['items' => $items]);          // arrays
return $this->json($entity, context: ['groups' => ['public']]);  // serializer groups
```

`$this->json()` uses the serializer if available, otherwise `json_encode`.

### Redirects

```php
return $this->redirectToRoute('app_user_show', ['id' => $user->getId()]);
return $this->redirect('https://example.com');
```

### Route groups (prefix + name + requirements at class level)

```php
#[Route('/api/v1', name: 'api_v1_', requirements: ['_locale' => 'en|nl'])]
final class ApiController extends AbstractController
{
    #[Route('/users', name: 'users_list', methods: ['GET'])]
    public function list(): Response { /* … */ }
}
```

Resulting route name: `api_v1_users_list`.

### Generating URLs

```php
$this->generateUrl('app_user_show', ['id' => 42]);
$this->generateUrl('app_user_show', ['id' => 42], UrlGeneratorInterface::ABSOLUTE_URL);
```

Or inject `UrlGeneratorInterface` outside controllers.

### Security on a route

```php
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[Route('/admin/users', methods: ['GET'])]
#[IsGranted('ROLE_ADMIN')]
public function admin(): Response { /* … */ }
```

Or per-method on an object:

```php
#[IsGranted('edit', 'user')]
public function edit(User $user): Response
```

### `make:controller`

```
bin/console make:controller HealthController
```

Creates the controller class, template (if applicable), and a stub test.

## Gotchas

- Always type-hint the request as `Request $request` (autowired). Do not
  call `Request::createFromGlobals()`.
- `__invoke` is fine for single-action controllers; multi-action is also
  fine — pick what reads better.
- Don't return arrays directly from controller methods — return `Response`
  / `JsonResponse`. Returning an array works only if the controller is
  decorated by `ControllerEvent` listeners that handle it (legacy).
- `$this->json()` on `AbstractController` is the most concise JSON
  response — prefer it over manual `new JsonResponse(...)`.
- Route names: default to `app_<controllerClassUnderscored>_<methodUnderscored>`
  if no `name:` is set.

## Docs

- Controllers: https://symfony.com/doc/7.4/controller.html
- Routing: https://symfony.com/doc/7.4/routing.html
- Param conversion: https://symfony.com/doc/7.4/doctrine.html#automatically-fetching-objects-mapentity
- Request payload mapping: https://symfony.com/doc/7.4/controller.html#mapping-request-data-to-typed-objects
