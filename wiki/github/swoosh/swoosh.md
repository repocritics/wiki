# swoosh/swoosh

> The de facto email library for Elixir — one composable `Email` struct, ~30
> provider adapters, and the testing story Phoenix ships by default.

[GitHub repo](https://github.com/swoosh/swoosh) ·
[Official docs](https://hexdocs.pm/swoosh) ·
[License: MIT](https://github.com/swoosh/swoosh/blob/main/LICENSE.txt)

## Overview

Swoosh is an email composition and delivery library for Elixir, started in
March 2016: a piped builder API over a plain `Swoosh.Email` struct, a
`Mailer` module you `use` into your app, and ~30 interchangeable adapters —
SMTP, Sendmail, and HTTP APIs for SendGrid, Mailgun, Postmark, Amazon SES,
Mailjet, Brevo, SparkPost, MS Graph, and more[^1]. Its ecosystem position
was cemented when Phoenix 1.6 (2021) adopted it as the default mailer in
project generators and `phx.gen.auth`, displacing Bamboo[^2] — most Elixir
apps created since then use Swoosh whether the author chose it or not.

The design tradeoff is deliberate minimalism: Swoosh is a *delivery*
library, not a delivery *pipeline*. It makes no assumptions about async
sending, queueing, retries, or rate limiting — `deliver/2` runs
synchronously and returns `{:ok, _}` or `{:error, _}`, full stop; the docs
punt background delivery to `Task` or a job queue like Oban[^3]. That keeps
the library small and predictable, but every production deployment ends up
building (or importing) the reliability layer itself.

The counterweight to that minimalism is an unusually complete testing story:
a `Local` adapter with an in-memory mailbox and web preview UI, a `Test`
adapter with `Swoosh.TestAssertions`, and a `Sandbox` adapter for
async/cross-process feature tests[^4] — the part competitors historically
lacked, and the main reason Phoenix standardized on it. At ~1.5k stars the
repo looks modest, but stars undersell generator-installed infrastructure:
it is actively maintained (pushed July 2026) and requires Elixir 1.16+ /
OTP 26+ on the current 1.26 line[^1].

## Getting Started

```elixir
# mix.exs: {:swoosh, "~> 1.26"} — plus {:gen_smtp, "~> 1.0"} for SMTP-based
# adapters, or an HTTP client (Hackney/Finch/Req) for API adapters

# config/config.exs
config :sample, Sample.Mailer,
  adapter: Swoosh.Adapters.Sendgrid,
  api_key: "SG.x.x"

# app code
defmodule Sample.Mailer do
  use Swoosh.Mailer, otp_app: :sample
end

defmodule Sample.UserEmail do
  import Swoosh.Email

  def welcome(user) do
    new()
    |> to({user.name, user.email})
    |> from({"Support", "support@example.com"})
    |> subject("Welcome!")
    |> html_body("<h1>Hello #{user.name}</h1>")
  end
end
# usage: Sample.UserEmail.welcome(user) |> Sample.Mailer.deliver()
```

## Architecture / How It Works

Four small pieces, loosely coupled:

- **`Swoosh.Email`** — a plain struct built with piped functions.
  Provider-specific features (templates, tags, metadata) go through
  `put_provider_option/3`, which each adapter maps to its API. The
  `Swoosh.Email.Recipient` protocol lets you `@derive` recipient conversion
  for your own structs so `to(%User{})` just works.
- **`Swoosh.Mailer`** — a `use` macro that reads adapter config from your
  OTP app env at delivery time; config can be overridden per-call.
- **Adapters** — modules implementing the `Swoosh.Adapter` behaviour
  (`deliver/2`, optionally `deliver_many/2`). API adapters translate the
  struct to each provider's JSON; SMTP-based ones (`SMTP`, `Sendmail`,
  `AmazonSES`) render MIME via `gen_smtp`. `Mua` is a pure-Elixir option.
- **`Swoosh.ApiClient`** — an HTTP-client behaviour decoupling adapters from
  the transport. Hackney is the default; Finch and Req work out of the box;
  it can be disabled entirely for SMTP/Local/Test-only setups[^1].

Development and test adapters are first-class: `Local` stores mail in an
in-memory mailbox served by `Plug.Swoosh.MailboxPreview` (Phoenix mounts it
at `/dev/mailbox`), `Test` sends deliveries as messages to the test process
for `assert_email_sent/1`, and `Sandbox` covers other processes[^4].

## Production Notes

- **No retries, no queue.** A provider 500 or timeout surfaces as
  `{:error, _}` once. Anything user-facing (signup, password reset) should
  go through Oban or similar with retry semantics; a bare `Task.start`
  loses the email on node restart or transient failure[^3].
- **`deliver_many/2` is inconsistent.** Batch delivery is optional per
  adapter and semantics differ by provider (one API call vs. a loop; partial
  failure reporting varies). Verify your adapter before relying on it.
- **Hackney is the default, not the best default.** Phoenix-generated apps
  usually already run Finch; configuring `Swoosh.ApiClient.Finch` avoids
  Hackney's dependency tree and its historical timeout/pool quirks.
- **Provider drift.** With ~30 adapters in one repo, less-popular adapters
  lag behind provider API changes; some are flagged "not fully featured"
  (Loops, PostUp)[^1]. `put_provider_option` keys are not portable — test
  against the real provider sandbox before switching.
- **SMTP means `gen_smtp`.** The SMTP adapter's TLS behavior and errors come
  from `gen_smtp`; OTP's tightening TLS defaults have broken connections to
  old relays over the years. `Mua` is the lighter-weight escape hatch.
- **Sandbox vs. async tests.** The `Test` adapter delivers to the calling
  process — emails sent from LiveView processes, Oban jobs, or Tasks never
  arrive in the test process. That is what `Sandbox` is for; using the
  wrong one is the most common Swoosh testing bug[^4].

## When to Use / When Not

**Use when:**
- You are on Elixir/Phoenix and need transactional email — it is the
  ecosystem default and the path of least resistance since Phoenix 1.6.
- You want provider portability: swapping SendGrid for Postmark is a
  config change plus a review of `provider_options`.
- You value the dev mailbox preview + test-assertion workflow.

**Avoid when:**
- You need built-in delivery guarantees, scheduling, or bulk campaign
  features — Swoosh sends one email when asked; everything else is on you.
- You are doing high-volume marketing/newsletter sends — provider-native
  campaign APIs fit better than a transactional struct-per-email model.
- You are deep in provider-specific features (dynamic templates, suppression
  management) — you will live in `put_provider_option` anyway, at which
  point the provider's own SDK or a thin Req client may be simpler.

## Alternatives

- beam-community/bamboo — the pre-2021 Elixir default, same adapter idea;
  use only when maintaining an app already built on it.
- gen-smtp/gen_smtp — Erlang SMTP client/server; use directly for raw SMTP
  control (custom relaying, receiving mail) rather than composition.
- ruslandoga/mua — minimal pure-Elixir SMTP client (also usable as a Swoosh
  adapter); use standalone when you want SMTP without `gen_smtp`.
- oban-bg/oban — not a mailer but the standard companion; use it *with*
  Swoosh whenever delivery must survive failures and restarts.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2016-03 | Initial public release; adapter model in place from the start. |
| 1.0 | 2020 | API stabilization on the 1.x line. |
| — | 2021-09 | Phoenix 1.6 adopts Swoosh in generators and `phx.gen.auth`[^2]. |
| 1.x | 2021– | `ApiClient` opened beyond Hackney; Finch and later Req supported out of the box[^1]. |
| 1.26 | 2026 | Current line; requires Elixir 1.16+ / OTP 26+; adapters added through 2025–2026 (Mailpit, ZeptoMail, Postal, Lettermint, Resend)[^1]. |

## References

[^1]: Swoosh README — adapters, requirements, ApiClient options. https://github.com/swoosh/swoosh
[^2]: Chris McCord, "Phoenix 1.6 released" — Swoosh-based mailer generators — 2021-09. https://www.phoenixframework.org/blog/phoenix-1.6-released
[^3]: Swoosh docs, "Async Emails" / Mailer — delivery is synchronous; background sending delegated to Task or a job library. https://hexdocs.pm/swoosh/Swoosh.Mailer.html
[^4]: Swoosh docs — `Swoosh.Adapters.Local`, `Swoosh.Adapters.Test`, `Swoosh.Adapters.Sandbox`, `Swoosh.TestAssertions`. https://hexdocs.pm/swoosh/Swoosh.Adapters.Test.html

## Tags

elixir, email, smtp, transactional-email, phoenix, mailer, adapter-pattern, sendgrid, mailgun, testing-tools
