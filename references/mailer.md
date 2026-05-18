# symfony/mailer

## When to use

You need to send email — transactional notifications, password resets, digest
emails, etc. Supports SMTP, plus transports for SendGrid, Mailgun, Postmark,
SES, Brevo, and others.

## Replaces

- PHPMailer
- SwiftMailer (EOL)
- Vendor SDKs called purely to send mail (SendGrid, Mailgun PHP SDK, etc.)
- Raw `mail()` calls

## Install

```
composer require symfony/mailer
# transport bridges (pick one):
composer require symfony/sendgrid-mailer
composer require symfony/mailgun-mailer
composer require symfony/amazon-mailer
```

Set `MAILER_DSN` in `.env` (e.g., `sendgrid+api://KEY@default`).

## Minimal example

```php
use Symfony\Bridge\Twig\Mime\TemplatedEmail;
use Symfony\Component\Mailer\MailerInterface;

public function __construct(private MailerInterface $mailer) {}

public function sendWelcome(string $to): void
{
    $email = (new TemplatedEmail())
        ->from('noreply@example.com')
        ->to($to)
        ->subject('Welcome!')
        ->htmlTemplate('emails/welcome.html.twig')
        ->context(['user' => $to]);

    $this->mailer->send($email);
}
```

## Common patterns

### Plain `Email` (no Twig)

```php
use Symfony\Component\Mime\Email;

$email = (new Email())
    ->from('noreply@example.com')
    ->to('alice@example.com')
    ->subject('Receipt')
    ->text('Plain body')
    ->html('<p>HTML body</p>');

$this->mailer->send($email);
```

### Attachments

```php
$email->attachFromPath('/path/to/invoice.pdf', 'invoice.pdf', 'application/pdf');
// or inline:
$email->embedFromPath('/path/to/logo.png', 'logo');
// reference in template as <img src="cid:logo">
```

### Async via Messenger (recommended for any user-facing flow)

In Flex projects `MAILER_DSN` is wired through Messenger automatically if a
`messenger.transport.async` is configured. Otherwise:

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async: '%env(MESSENGER_TRANSPORT_DSN)%'
        routing:
            'Symfony\Component\Mailer\Messenger\SendEmailMessage': async
```

The mailer then enqueues `SendEmailMessage`; a worker consumes and sends.

### Multiple mailer transports

```yaml
framework:
    mailer:
        transports:
            main: '%env(MAILER_DSN)%'
            newsletter: '%env(NEWSLETTER_DSN)%'
```

Pick at send time:

```php
$email->getHeaders()->addTextHeader('X-Transport', 'newsletter');
```

## Gotchas

- `TemplatedEmail` requires `symfony/twig-bundle`.
- `from()` is mandatory; configure a project-wide default via
  `framework.mailer.envelope` to avoid setting it everywhere.
- In dev, use `MAILER_DSN=null://null` to swallow mail, or point at
  Mailpit/MailHog for inspection.
- For DKIM/SPF you must configure the transport's account — Symfony just
  hands the message off.

## Docs

https://symfony.com/doc/7.4/mailer.html
