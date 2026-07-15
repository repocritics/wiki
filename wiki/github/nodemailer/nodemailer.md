# nodemailer/nodemailer

> Zero-dependency SMTP client for Node.js — it composes and sends RFC 822 messages; it is not a mail server.

[GitHub repo](https://github.com/nodemailer/nodemailer) ·
[Official website](https://nodemailer.com/) ·
License: MIT-0[^1]

## Overview

Nodemailer is the default answer to "how do I send an email from Node.js." It has
existed since 2011[^2] and for most of that time has been effectively unrivalled
in its niche: a single library that composes a MIME message and hands it to an
SMTP server. As of this writing it carries ~17.6k stars and, notably, **zero
runtime dependencies** — the SMTP client, MIME composer, DKIM signer, and OAuth2
handling are all vendored in-tree.

The defining thing to understand is what Nodemailer is *not*. It is a mail
*client*, not a mail *transfer agent*. It opens a connection to an SMTP relay you
provide, streams a message, and reports whether the relay accepted it. It does
not queue, retry, handle bounces, warm IPs, or manage deliverability. "The server
accepted the message" and "the message reached the inbox" are entirely different
claims, and Nodemailer only speaks to the first. Teams repeatedly discover this
the hard way when mail lands in spam or silently disappears — that is not a
Nodemailer bug, it is the boundary of what the library does.

The project is essentially the work of a single maintainer, Andris Reinman, who
also builds a family of commercial and open email infrastructure (EmailEngine,
ZoneMTA, WildDuck). The README markets EmailEngine directly[^3]. This is worth
knowing: the library is stable and widely trusted, but bus-factor and roadmap are
concentrated in one person.

## Getting Started

```bash
npm install nodemailer
# TypeScript users also want the community types (not official):
npm install -D @types/nodemailer
```

```js
const nodemailer = require("nodemailer");

const transporter = nodemailer.createTransport({
  host: "smtp.example.com",
  port: 587,            // STARTTLS — set secure:false here
  secure: false,        // true ONLY for port 465 (implicit TLS)
  auth: { user: "postmaster@example.com", pass: process.env.SMTP_PASS },
});

const info = await transporter.sendMail({
  from: '"App" <no-reply@example.com>',
  to: "user@example.com",
  subject: "Hello",
  text: "Plaintext body",
  html: "<b>HTML body</b>",
});

console.log(info.messageId, info.accepted, info.rejected);
```

## Architecture / How It Works

`createTransport()` returns a transport object; the API is deliberately
plugin-shaped. The built-in transports are:

- **SMTP** (default) — `smtp-connection` under the hood, optionally pooled.
- **`pool: true`** — `smtp-pool`, which keeps a bounded set of open connections
  and reuses them (`maxConnections`, `maxMessages`, `rateLimit`).
- **sendmail** — pipe to a local `sendmail` binary.
- **SES** — thin wrapper that hands the composed raw message to the AWS SDK.
- **stream / JSON** — do not send at all; emit the built message for tests.

Message construction goes through **MailComposer**, which builds the MIME tree
(multipart/alternative for text+html, multipart/mixed for attachments, embedded
images via `cid:` references) and streams it rather than buffering the whole
message in memory — relevant for large attachments. **DKIM signing** is built in
(`dkim` option) and happens as the message streams. **OAuth2** is a first-class
auth type with automatic access-token refresh, which is how you talk to Gmail and
Microsoft 365 now that basic auth is deprecated on both.

DNS resolution is a subtle internal detail: Nodemailer resolves the SMTP host via
Node's `dns.resolve4()`/`resolve6()` (c-ares), falling back to `dns.lookup()`.
Because c-ares does not consult the OS resolver, custom `/etc/hosts` entries and
some split-horizon DNS setups are ignored unless you hard-code the IP[^4].

## Production Notes

- **It is not an ESP.** No bounce handling, no suppression lists, no open/click
  tracking, no deliverability engineering. For anything at scale or where inbox
  placement matters, Nodemailer is the *transport*, not the solution — pair it
  with a real relay (SES, a dedicated MTA) or use an ESP API instead.
- **No built-in retry or queue.** `sendMail` either resolves or rejects once. If
  you need durability across process restarts, put a job queue in front of it.
- **Gmail is a known pain point.** The maintainer's own guidance is blunt: Gmail
  "either works well, or it does not work at all," and the recommendation is to
  switch providers rather than debug it[^3]. App Passwords or OAuth2 are required;
  plain-password auth is effectively dead.
- **TLS `secure` flag is the #1 misconfiguration.** `secure: true` is only for
  port 465. For 587/25 use `secure: false` and let Nodemailer STARTTLS-upgrade;
  setting `secure:false` does *not* mean unencrypted.
- **Pooling requires tuning.** Under load, tune `maxConnections`, `maxMessages`,
  and `rateLimit` to your relay's limits, or you will get throttled/disconnected.
- **TypeScript support is unofficial.** Types live in the community
  `@types/nodemailer` package; type mismatches are not the maintainer's problem
  per the README. Node.js is the only officially supported runtime.
- **Deliverability (SPF/DKIM/DMARC) is on you.** Nodemailer can DKIM-sign, but
  aligning DNS records and reputation is entirely outside the library.

## When to Use / When Not

**Use when:**
- You send transactional email from Node and control an SMTP relay.
- You need full control over the MIME message: attachments, embedded images,
  calendar invites, custom headers, DKIM signing.
- You want OAuth2-based sending to Gmail/Microsoft 365.
- You want zero dependencies and a stable, boring, well-understood library.

**Avoid / augment when:**
- You need deliverability, bounce processing, analytics, or suppression — use an
  ESP API instead of (or in front of) raw SMTP.
- You're sending bulk/marketing mail — this is not a campaign engine.
- You need durable queuing and retries — add that layer yourself.
- You're on a non-Node runtime (Deno/Bun/edge) — support is not guaranteed.

## Alternatives

- resend/resend-node — use when you want a modern ESP HTTP API with deliverability, bounces, and dashboards handled for you instead of raw SMTP.
- sendgrid/sendgrid-nodejs — use when you want an established ESP with templates, analytics, and a large free tier.
- postalsys/emailengine — same author; use when you also need to *receive* mail and want IMAP/SMTP exposed over REST with webhooks and OAuth2.
- eleith/emailjs — use when you want a smaller SMTP-only client without Nodemailer's transport-plugin surface.
- aws/aws-sdk-js-v3 (SES `SendEmailCommand`) — use when you're already on AWS and want to send through SES directly.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-01 | Repository created; early SMTP sending library[^2]. |
| 3.x | 2016–2017 | Major rewrite around `createTransport` + pluggable transports, streaming MailComposer, built-in DKIM and calendar support. |
| 6.0 | 2019 | Promise/async-first API, dropped legacy Node support, zero-dependency posture solidified. |
| — | ongoing | Long-lived 6.x line; OAuth2 refresh, SES v3 wrapper, MIT-0 relicensing[^1]. |

## References

[^1]: README, "License": Nodemailer is licensed under the MIT No Attribution (MIT-0) license. GitHub's license detector reports NOASSERTION because it does not recognize the MIT-0 file variant. https://github.com/nodemailer/nodemailer
[^2]: GitHub API `repos/nodemailer/nodemailer` — `created_at` 2011-01-19. https://github.com/nodemailer/nodemailer
[^3]: Nodemailer README, "Having an issue?" (Gmail guidance) and EmailEngine promotion. https://github.com/nodemailer/nodemailer/blob/master/README.md
[^4]: Nodemailer README, "I have issues with DNS / hosts file" — c-ares resolution via `dns.resolve4/6` with `dns.lookup` fallback. https://github.com/nodemailer/nodemailer/blob/master/README.md

## Tags

javascript, nodejs, email, smtp, mime, transactional-email, dkim, oauth2, mail-client, zero-dependency
