# PHPMailer/PHPMailer

> The classic PHP email library — a thin, battle-hardened wrapper over SMTP and MIME that predates Composer and still ships in half the PHP web.

[GitHub repo](https://github.com/PHPMailer/PHPMailer) ·
[Packagist](https://packagist.org/packages/phpmailer/phpmailer) ·
[License: LGPL-2.1](https://github.com/PHPMailer/PHPMailer/blob/master/LICENSE)

## Overview

PHPMailer is the oldest widely-used email library for PHP. It was originally written in 2001 by Brent R. Matzelle as a SourceForge project, taken over by Marcus Bointon and Andy Prevost in 2004, and consolidated onto GitHub as the canonical repo in 2013[^1]. Its job is narrow: assemble a correctly-formatted MIME message (headers, encodings, multipart bodies, attachments, DKIM/S/MIME signatures) and deliver it over SMTP, `sendmail`, `qmail`, or PHP's native `mail()`. It does not manage queues, templates, retries, bounces, or deliverability — those are your problem or another library's.

Its reach is the story. PHPMailer is a bundled dependency of WordPress, Drupal, Joomla, SugarCRM, Yii, and thousands of other projects[^2], which means it runs on a very large fraction of all PHP sites whether or not the developer chose it. That install base is also why its security history matters more than its feature set: a bug in PHPMailer is a bug in a meaningful slice of the internet, and the 2016–2017 remote-code-execution chain (see Production Notes) was treated as an internet-scale event.

The defining tension is age versus modernity. The API is imperative and mutable — you `new` an object, set public properties (`$mail->Host`, `$mail->Subject`), call `addAddress()`, then `send()`. It is not immutable, not PSR-anything, and not fluent. Newer libraries (Symfony Mailer, Laminas Mail) offer transports, message objects, and DI-friendly design. PHPMailer's counter-argument is that it is a single small dependency with almost no transitive baggage, compatible from PHP 5.5 through 8.5, and that the awkward object has been debugged by millions of users for two decades.

## Getting Started

```sh
composer require phpmailer/phpmailer
```

```php
<?php
use PHPMailer\PHPMailer\PHPMailer;
use PHPMailer\PHPMailer\Exception;

require 'vendor/autoload.php';

$mail = new PHPMailer(true); // true => throw exceptions on error

try {
    $mail->isSMTP();
    $mail->Host       = 'smtp.example.com';
    $mail->SMTPAuth   = true;
    $mail->Username   = 'user@example.com';
    $mail->Password   = 'secret';
    $mail->SMTPSecure = PHPMailer::ENCRYPTION_SMTPS; // implicit TLS
    $mail->Port       = 465;                          // use 587 for STARTTLS

    $mail->setFrom('from@example.com', 'Mailer');
    $mail->addAddress('joe@example.net', 'Joe User');
    $mail->addAttachment('/var/tmp/file.tar.gz');

    $mail->isHTML(true);
    $mail->Subject = 'Here is the subject';
    $mail->Body    = 'HTML message body <b>in bold!</b>';
    $mail->AltBody = 'Plain-text body for non-HTML clients';

    $mail->send();
} catch (Exception $e) {
    echo "Mailer Error: {$mail->ErrorInfo}";
}
```

Note the two-flavour error model: `new PHPMailer(true)` throws `Exception`, while `new PHPMailer()` (or `false`) makes `send()` return `false` and leaves the reason in `$mail->ErrorInfo`. Mixing the two up is the most common beginner bug.

## Architecture / How It Works

PHPMailer is deliberately tiny — the runtime is effectively three classes under `PHPMailer\PHPMailer`:

- **`PHPMailer`** — the message builder and orchestrator. Holds all state as public properties, does MIME assembly, address validation, encoding selection (7bit/8bit/base64/quoted-printable), attachment embedding, and header construction including the header-injection defenses that make it safer than hand-rolled `mail()` calls.
- **`SMTP`** — a from-scratch SMTP client. Speaks EHLO/AUTH/STARTTLS/DATA directly over a stream socket; supports `LOGIN`, `PLAIN`, `CRAM-MD5`, and `XOAUTH2` mechanisms. `SMTPDebug` exposes the raw protocol conversation, which is the primary debugging tool.
- **`Exception`** — a namespaced exception type, loaded even if you don't use exceptions because it's referenced internally.

Optional pieces (`OAuth`, `POP3`) are separate files pulled in only when needed. XOAUTH2 depends on `league/oauth2-client` plus a service adapter; that is the one place PHPMailer reaches for an external dependency, and it's opt-in.

There is no transport abstraction layer in the modern sense. `isSMTP()`, `isMail()`, `isSendmail()`, and `isQmail()` flip an internal mode flag rather than swapping a strategy object. This keeps the surface small but means you can't cleanly inject a fake transport for testing — tests typically point SMTP at a local catch-all server (e.g. MailHog/Mailpit) instead of mocking.

Because state lives in mutable public properties, reusing one instance across a mailing list requires manually clearing recipients with `clearAddresses()` / `clearAttachments()` between sends, or you will re-send to accumulated recipients. This footgun is called out directly in the project's own examples.

## Production Notes

**The security history is the headline.** PHPMailer's ubiquity made it a high-value target, and it has a real CVE record you must respect:

- **CVE-2016-10033 / CVE-2016-10045** — remote code execution via the `Sender` address when using the `mail()` transport: attacker-controlled email addresses were passed unsanitised into `sendmail`'s `-f` argument, enabling shell option injection (Dawid Golunski's "PwnScriptum")[^3]. Fixed in 5.2.18/5.2.20. This alone is the reason to prefer SMTP-to-localhost over the `mail()` transport.
- **CVE-2017-5223** — local file disclosure through `msgHTML()`/`addAttachment()` path handling.
- **CVE-2020-13625** — an `escapeshellarg` bypass on some platforms, again in the `mail()` path.

The practical takeaway: **avoid the `mail()` transport, use `isSMTP()` to localhost, and keep the library patched.** An outdated bundled copy inside a CMS plugin is the classic exposure.

**It is a builder, not a mailer service.** No queueing, no async, no retry/backoff, no rate limiting, no bounce or complaint handling, no template engine. `send()` is synchronous and blocks for the full SMTP round-trip; sending to a large list in a web request will time out. Production senders wrap it in a queue/worker (or use a provider API) and treat PHPMailer purely as the RFC-correct message constructor.

**Deliverability is on you.** PHPMailer will DKIM-sign if you configure a key, but SPF/DMARC/DNS, IP reputation, and provider relationships are entirely outside its scope. Sending directly from an app server's `mail()` or a raw SMTP connection to recipients' MX hosts generally lands in spam; most teams relay through SES/SendGrid/Postmark/Mailgun SMTP endpoints and let PHPMailer just build and hand off the message.

**Debugging.** Nearly every support question ("SMTP Error: Could not connect to SMTP host") is a TLS, port, firewall, or credential issue, not a library bug. Set `$mail->SMTPDebug = SMTP::DEBUG_SERVER` to see the raw conversation before filing anything.

**Upgrade pain is mostly the 5.2 → 6.0 jump.** 6.0 introduced the `PHPMailer\PHPMailer` namespace and moved sources into `src/`; upgrading from the un-namespaced 5.2 line requires code changes (`use` statements, class references) and is not a drop-in bump[^4]. Within the 6.x and 7.x lines, upgrades have been comparatively smooth. The 5.2 branch is end-of-life and receives no security fixes.

## When to Use / When Not

**Use when:**
- You need one small, low-dependency library to build and send correct MIME email from PHP.
- You're on legacy PHP (5.5+) or maintaining a codebase already standardised on it.
- You want attachments, inline images, HTML+plain multipart, DKIM/S-MIME, and SMTP auth without assembling them yourself.
- You're relaying through a provider's SMTP endpoint and just need a reliable message builder.

**Avoid when:**
- You want a modern, testable, transport-abstracted API with immutable message objects — reach for Symfony Mailer.
- You need queueing, async delivery, retries, or bounce handling — that's an application/provider concern PHPMailer doesn't cover.
- You can use a Symfony/Laravel stack whose framework already ships a mailer component (Laravel's `Mail`/`Mailable` sits on Symfony Mailer).
- You'd be tempted to use the `mail()` transport with any user-influenced address — the security history argues against it.

## Alternatives

- symfony/mailer — modern successor with pluggable transports, immutable messages, and provider bridges (SES/SendGrid/etc.); use when you want a testable, DI-friendly design on PHP 8.
- laminas/laminas-mail — the Zend-lineage mail component; use when you're already in the Laminas ecosystem.
- zetacomponents/Mail — long-standing standalone MIME/mail library; use when you want an alternative builder outside the Symfony world.
- swiftmailer/swiftmailer — the former go-to; abandoned and folded into Symfony Mailer, so use only for legacy maintenance, not new work.
- php-native mail() — built in and dependency-free, but unsafe and featureless; the reason PHPMailer exists.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x (SF) | 2001 | Original SourceForge project by Brent R. Matzelle[^1]. |
| — | 2004 | Marcus Bointon and Andy Prevost take over maintenance[^1]. |
| — | 2013 | GitHub PHPMailer org becomes the canonical repo[^1]. |
| 5.2 | — | Pre-namespace line, PHP 5.0–7.0; now end-of-life, no security fixes. |
| 6.0 | 2017 | `PHPMailer\PHPMailer` namespace, sources moved to `src/`, PHP 5.5+[^4]. |
| 7.0 | 2026 | Current major line; PHP 5.5 through 8.5 compatibility[^2]. |

## References

[^1]: PHPMailer README, "History" section — SourceForge origins (2001), maintainer handover (2004), GitHub consolidation (2013). https://github.com/PHPMailer/PHPMailer#history
[^2]: PHPMailer README, "Features" — bundled by WordPress, Drupal, Joomla, etc.; PHP 5.5–8.5 support; Composer install `^7.0.0`. https://github.com/PHPMailer/PHPMailer#readme
[^3]: Dawid Golunski, "PwnScriptum" — PHPMailer RCE (CVE-2016-10033). https://exploitbox.io/vuln/PHPMailer-Exploit-Remote-Code-Exec-CVE-2016-10033-Vuln.html
[^4]: PHPMailer upgrade guide — 5.2 to 6.0 namespace/`src/` migration. https://github.com/PHPMailer/PHPMailer/blob/master/UPGRADING.md

## Tags

php, email, smtp, mime, mailer, php-library, attachments, dkim, xoauth2, security-sensitive
