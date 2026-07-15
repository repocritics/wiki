# knadh/listmonk

> Self-hosted newsletter and mailing-list manager: one Go binary, a PostgreSQL database, and your own SMTP server.

[GitHub repo](https://github.com/knadh/listmonk) ·
[Official website](https://listmonk.app) ·
[License: AGPL-3.0](https://github.com/knadh/listmonk/blob/master/LICENSE)

## Overview

listmonk is a standalone mailing-list and newsletter application: subscriber
lists, campaign composition, scheduled/queued sending, bounce processing, and a
transactional-email API, all behind a Vue admin dashboard. It ships as a single
Go binary that embeds its frontend assets and talks to a PostgreSQL database —
there is no application runtime to install beyond Postgres itself[^1]. The
project was started by Kailash Nadh (CTO of Zerodha) in 2019 and has grown into
one of the most-deployed open-source alternatives to Mailchimp-style hosted
newsletter services[^2].

The defining tradeoff is that listmonk is the *application*, not the *mail
infrastructure*. It composes, queues, and hands messages to SMTP servers you
provide (self-run, AWS SES, Postmark, SendGrid, etc.), but it does not deliver
mail itself, manage IP reputation, or handle DNS/SPF/DKIM. Teams who expect a
turnkey "send email" product are surprised by how much of deliverability lives
outside listmonk's scope. In return you get a fast, scriptable, fully-owned
sending pipeline with no per-subscriber SaaS pricing. A second consequence of
that philosophy: segmentation is raw SQL — subscriber targeting is expressed as
`WHERE` fragments against Postgres, which makes arbitrarily complex queries
trivial and lets an operator scan millions of rows. Power and footgun at once.

## Getting Started

```shell
# Docker (recommended): pull the sample compose file and start.
curl -LO https://github.com/knadh/listmonk/raw/master/docker-compose.yml
docker compose up -d
# Admin UI on http://localhost:9000
```

```shell
# Binary install:
./listmonk --new-config     # writes config.toml — edit DB + admin creds
./listmonk --install        # creates and seeds the Postgres schema
./listmonk                  # serve; --upgrade applies migrations idempotently
```

Campaigns are authored in the dashboard (or the REST API) using Go
`text/template` + HTML, with per-subscriber attributes available in the template
context. Transactional sends go through `POST /api/tx` with a template ID and
recipient data.

## Architecture / How It Works

listmonk is a monolith by design. The Go backend serves both the JSON API and
the embedded Vue/Buefy SPA; all persistent state lives in PostgreSQL[^1].

- **SQL-first data layer.** Queries are defined as named statements in a
  `queries.sql` file (goyesql-style) rather than an ORM. This keeps the hot
  paths as hand-written SQL and makes list segmentation a first-class SQL
  concept: a "segment" is a stored `WHERE` expression evaluated against the
  `subscribers` table.
- **Campaign worker.** A background manager pulls running campaigns, batches
  subscribers, renders each message from the campaign template, and pushes them
  to the configured messenger. Concurrency and per-second message rate are
  config-tunable to respect provider throttles.
- **Messengers.** The default messenger is SMTP, with support for multiple SMTP
  servers used concurrently. An email messenger pool spreads load; there is also
  a generic HTTP/webhook messenger for SMS/other gateways.
- **Bounce processing.** listmonk can scan a POP3/IMAP mailbox for bounce
  messages, or ingest bounce webhooks from SES/SendGrid, and mark or delete
  subscribers past a threshold.
- **Media.** Uploaded assets go to the local filesystem by default, or to an
  S3-compatible bucket — the latter is required for any multi-container setup.
- **Settings split.** Startup essentials (DB DSN, admin bind) live in
  `config.toml`; most operational settings (SMTP servers, appearance, privacy)
  are stored in the database and edited from the UI.

Newer releases added a multi-user auth layer with roles/permissions and OIDC
single sign-on, replacing the original single-admin model[^3]. Dashboard
aggregate counts are served from a materialized view refreshed periodically
rather than counted live, which keeps the dashboard cheap on large datasets.

## Production Notes

- **Postgres is the whole system.** listmonk itself is close to stateless; your
  scaling, backup, and HA story *is* your Postgres story. Large deployments
  (millions of subscribers) live or die on Postgres tuning, autovacuum, and
  index health, not on the Go process.
- **SQL segments are unbounded.** Because a query segment is arbitrary SQL, a
  poorly written targeting expression can table-scan the subscribers table and
  stall sending. Review campaign queries the way you'd review a production DB
  query.
- **Not horizontally scalable for sending.** The campaign worker assumes a
  single active instance driving a campaign. Running multiple app instances
  against one database to parallelize a single campaign is not a supported
  design and risks duplicate sends; scale sending by adding SMTP servers /
  raising concurrency, not app replicas.
- **Deliverability is on you.** SPF/DKIM/DMARC, warm-up, IP reputation, and
  bounce/complaint handling are operational responsibilities outside listmonk.
  Most successful deployments front listmonk with a reputable SMTP relay rather
  than raw port-25 sending.
- **Media on local disk breaks containers.** The default filesystem media store
  does not survive container replacement; switch to S3 before going multi-node
  or ephemeral.
- **Upgrades are migration-driven.** `./listmonk --upgrade` (or the Docker
  equivalent) applies schema migrations idempotently; back up Postgres first and
  read release notes for any manual steps.

## When to Use / When Not

**Use when:**
- You want to own your subscriber data and sending pipeline without SaaS
  per-contact pricing.
- You already have (or will run) a reliable SMTP relay and just need the
  application layer on top.
- You value SQL-based segmentation and a scriptable REST API over drag-and-drop
  marketing automation.
- You want a single binary + Postgres, not a multi-service stack.

**Avoid when:**
- You expect turnkey deliverability — listmonk sends *through* SMTP, it is not a
  mail server and won't manage reputation.
- You need CRM-grade marketing automation: lead scoring, multi-branch drip
  workflows, sales integrations.
- You cannot operate PostgreSQL responsibly at your list size.
- You need managed hosting with SLAs rather than self-hosting.

## Alternatives

- mautic/mautic — PHP marketing-automation suite with CRM, lead scoring, and
  visual campaign builders; use when you need automation workflows, not just
  newsletters.
- mailtrain-org/mailtrain — Node.js self-hosted newsletter app; use when you
  prefer the Node ecosystem and visual automation.
- keila-io/keila — Elixir self-hosted newsletter tool; use when you want a
  simpler, more opinionated single-purpose option.
- phpList/phplist3 — long-established PHP mailing-list manager; use for a
  low-footprint LAMP-stack deployment.
- postalserver/postal — a full self-hosted SMTP mail server; use *alongside*
  something like listmonk when you need the delivery infrastructure listmonk
  deliberately does not provide.

## History

| Version | Date | Notes |
|---------|------|-------|
| Repo created | 2019-06-26 | Started by Kailash Nadh; Go backend + Vue/Buefy frontend, single binary[^2]. |
| 1.0.0 | 2021 | First stable release after a long public beta. |
| 2.0.0 | 2022 | Major feature release; expanded campaign and messenger capabilities. |
| 3.0.0 | 2024 | Multi-user authentication, roles/permissions, and OIDC SSO[^3]. |

## References

[^1]: listmonk README and project description — self-hosted, single-binary Go app backed by PostgreSQL. https://github.com/knadh/listmonk
[^2]: listmonk official site and documentation. https://listmonk.app
[^3]: listmonk documentation, user management / roles / OIDC. https://listmonk.app/docs

## Tags

go, vue, newsletter, mailing-list, email-marketing, smtp, self-hosted, postgresql, transactional-email, agpl
