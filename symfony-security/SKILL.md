---
name: symfony-security
description: "Symfony security - firewalls, authenticators, voters, access control, password hashing, #[IsGranted]. Use for any authentication or authorization task in a Symfony project - login, API tokens, roles, permissions, securing endpoints or routes."
---

# Security — firewalls, authenticators, voters

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need authentication (who is the user?) or authorization (can they do
this?) — login forms, API token auth, role checks, per-object permissions.

## What you need

```
composer require security
composer require maker --dev   # for make:user, make:auth, make:voter
```

## Set up a user

```
bin/console make:user
```

Creates the `User` entity (PSR-4 under `App\Entity\User`), implementing
`UserInterface` + `PasswordAuthenticatedUserInterface`, and the matching
`UserRepository`.

## Set up auth

```
bin/console make:auth
```

Choose form-login (web) or API-token-style. Generates the authenticator
class and wires `config/packages/security.yaml`.

## Minimal `security.yaml`

```yaml
security:
    password_hashers:
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email
    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false
        main:
            lazy: true
            provider: app_user_provider
            form_login:
                login_path: app_login
                check_path: app_login
                enable_csrf: true
            logout:
                path: app_logout
            remember_me:
                secret: '%kernel.secret%'
                lifetime: 604800
    access_control:
        - { path: ^/admin, roles: ROLE_ADMIN }
        - { path: ^/profile, roles: ROLE_USER }
```

## Common patterns

### Restrict a route or action

```php
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[Route('/admin', methods: ['GET'])]
#[IsGranted('ROLE_ADMIN')]
public function dashboard(): Response { /* … */ }
```

### Get the current user

```php
public function whoami(#[CurrentUser] ?User $user): Response
{
    return $this->json(['user' => $user?->getUserIdentifier()]);
}
```

The `#[CurrentUser]` attribute gives a typed user (or `null` if anonymous).
Inside `AbstractController`, `$this->getUser()` returns the same.

### Hash a password

```php
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

public function __construct(private UserPasswordHasherInterface $hasher) {}

$user->setPassword($this->hasher->hashPassword($user, $plaintext));
```

### Voter — per-object permissions

```
bin/console make:voter PostVoter
```

```php
namespace App\Security\Voter;

use App\Entity\Post;
use App\Entity\User;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;

final class PostVoter extends Voter
{
    public const EDIT = 'POST_EDIT';
    public const VIEW = 'POST_VIEW';

    protected function supports(string $attribute, mixed $subject): bool
    {
        return in_array($attribute, [self::EDIT, self::VIEW], true)
            && $subject instanceof Post;
    }

    protected function voteOnAttribute(
        string $attribute,
        mixed $subject,
        TokenInterface $token
    ): bool {
        $user = $token->getUser();
        if (!$user instanceof User) {
            return false;
        }

        return match ($attribute) {
            self::VIEW => $subject->isPublished() || $subject->getAuthor() === $user,
            self::EDIT => $subject->getAuthor() === $user,
        };
    }
}
```

Use:

```php
#[IsGranted('POST_EDIT', subject: 'post')]
public function edit(Post $post): Response
```

Or imperatively:

```php
$this->denyAccessUnlessGranted('POST_EDIT', $post);
```

### Custom authenticator (API token)

```
bin/console make:auth   # pick "Custom"
```

Implement `Symfony\Component\Security\Http\Authenticator\AbstractAuthenticator`:

```php
public function supports(Request $request): ?bool
{
    return $request->headers->has('X-API-TOKEN');
}

public function authenticate(Request $request): Passport
{
    $token = $request->headers->get('X-API-TOKEN');
    return new SelfValidatingPassport(
        new UserBadge($token, fn ($t) => $this->userRepo->findByApiToken($t))
    );
}
```

### Programmatic login (post-registration)

```php
use Symfony\Bundle\SecurityBundle\Security;

$this->security->login($user, 'form_login', 'main');
```

### Switch user (impersonation)

```yaml
firewalls:
    main:
        switch_user: { role: ROLE_ADMIN }
```

Then visit `/?_switch_user=alice@example.com` (admins only).

## Gotchas

- Roles must start with `ROLE_`. `ROLE_ADMIN` is a string, not an enum —
  use constants for typo safety.
- `is_granted()` (Twig) and `IS_AUTHENTICATED_FULLY` checks: anonymous
  visitors are *not* `ROLE_USER`. Use `IS_AUTHENTICATED_FULLY` for
  "logged-in" gates, `PUBLIC_ACCESS` for "always allow".
- `access_control` is evaluated top-to-bottom — order matters. More
  specific paths first.
- Don't store sensitive data in `User` session — keep what
  `serialize()`/`__serialize()` exposes minimal.
- For API auth use `firewalls.api: { stateless: true }`. Stateful sessions
  on APIs cause CSRF surprises.
- Password hashers: `'auto'` is right for new code. Don't pick `bcrypt`
  explicitly unless you need backward compat.

## Docs

- Security overview: https://symfony.com/doc/7.4/security.html
- Authenticators: https://symfony.com/doc/7.4/security/custom_authenticator.html
- Voters: https://symfony.com/doc/7.4/security/voters.html
- Password hashing: https://symfony.com/doc/7.4/security/passwords.html
