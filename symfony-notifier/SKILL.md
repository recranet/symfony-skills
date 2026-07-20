---
name: symfony-notifier
description: Send notifications with symfony/notifier - chat (Slack, Discord, Telegram), SMS (Twilio, Vonage), push. Use whenever a Symfony project sends a notification to a chat channel, phone, or device, even if the user reaches for a vendor SDK.
---

# symfony/notifier

> If the project contains `.ddev/config.yaml`, prefix every shell command
> with `ddev` (full mapping in the `symfony` skill's `references/ddev.md`).
> Examples target Symfony 7.4, 8.0 and newer — verify the installed version before
> using newer features.

## When to use

You need to deliver non-email notifications: SMS, push, chat (Slack/Discord/
Teams/Telegram), or browser push. Notifier abstracts over many providers.

## Replaces

- Twilio PHP SDK (for sending SMS only)
- Slack/Discord/Telegram SDKs (for posting messages only)
- Vendor push SDKs (Expo, FCM, OneSignal) when used only to send

## Install

```
composer require symfony/notifier
# transport bridges:
composer require symfony/twilio-notifier
composer require symfony/slack-notifier
composer require symfony/discord-notifier
composer require symfony/telegram-notifier
```

Configure DSNs in `.env`:

```
TWILIO_DSN=twilio://SID:TOKEN@default?from=+15551234567
SLACK_DSN=slack://TOKEN@default?channel=alerts
```

## Minimal example

Send a chat message:

```php
use Symfony\Component\Notifier\ChatterInterface;
use Symfony\Component\Notifier\Message\ChatMessage;

public function __construct(private ChatterInterface $chatter) {}

$this->chatter->send(new ChatMessage('Deploy finished'));
```

Send an SMS:

```php
use Symfony\Component\Notifier\TexterInterface;
use Symfony\Component\Notifier\Message\SmsMessage;

public function __construct(private TexterInterface $texter) {}

$this->texter->send(new SmsMessage('+15551112222', 'Your code is 123456'));
```

## Common patterns

### Configure transports + channels

```yaml
# config/packages/notifier.yaml
framework:
    notifier:
        chatter_transports:
            slack: '%env(SLACK_DSN)%'
            discord: '%env(DISCORD_DSN)%'
        texter_transports:
            twilio: '%env(TWILIO_DSN)%'
        channel_policy:
            urgent: ['chat/slack', 'sms/twilio']
            medium: ['chat/slack']
            low:    ['chat/discord']
        admin_recipients:
            - { email: ops@example.com, phone: '+15551112222' }
```

### Full `Notification` with recipients + importance

```php
use Symfony\Component\Notifier\NotifierInterface;
use Symfony\Component\Notifier\Notification\Notification;
use Symfony\Component\Notifier\Recipient\Recipient;

public function __construct(private NotifierInterface $notifier) {}

$notification = (new Notification('Deploy failed', ['chat/slack', 'sms/twilio']))
    ->content('Build #123 failed on main')
    ->importance(Notification::IMPORTANCE_URGENT);

$this->notifier->send($notification, new Recipient(email: 'ops@example.com', phone: '+15551112222'));
```

### Provider-specific options (e.g., Slack blocks)

```php
use Symfony\Component\Notifier\Bridge\Slack\SlackOptions;
use Symfony\Component\Notifier\Bridge\Slack\Block\SlackSectionBlock;

$options = (new SlackOptions())
    ->block((new SlackSectionBlock())->text('*Deploy* finished'));

$message = (new ChatMessage('Deploy finished'))->options($options);
$this->chatter->send($message);
```

## Gotchas

- A transport DSN that includes `?channel=…` makes channel optional on each
  message — handy for fixed-target alerting.
- `Chatter`/`Texter`/`Notifier` are separate interfaces; pick the narrowest
  one that fits.
- For high volume or retry semantics, route Notifier messages through
  Messenger (`framework.notifier.message_bus` config).
- `Notification::IMPORTANCE_*` only matters if you defined a `channel_policy`.

## Docs

https://symfony.com/doc/7.4/notifier.html
