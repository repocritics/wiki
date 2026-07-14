# getsentry/sentry

> Server-side of the Sentry error-tracking and performance-monitoring platform — a Django/React monolith fronted by a Rust ingestion layer and backed by ClickHouse.

[GitHub repo](https://github.com/getsentry/sentry) ·
[Official website](https://sentry.io) ·
License: FSL-1.1-Apache-2.0 (Fair Source)[^1]

## Overview

Sentry is application-monitoring software: SDKs embedded in your app capture exceptions, stack traces, and performance spans and ship them to a Sentry server, which groups them into "issues," deduplicates, alerts, and renders them for debugging. This repository is the *server* — the ingestion pipeline, storage/query layer, and web UI. The thing most developers touch day-to-day is one of the ~25 language SDKs (sentry-python, sentry-javascript, sentry-java, …), which live in separate repos; this repo is what those SDKs talk to, whether via sentry.io (SaaS) or a self-hosted install[^2].

Sentry began in 2008 as a Django logging side-project by David Cramer and was open-sourced early; the company (Sentry / Functional Software Inc.) formed in 2012[^3]. For most of its history the codebase carried a BSD license. That changed twice: to the Business Source License (BSL) in 2019, then to the Functional Source License (FSL) in November 2023[^1]. FSL is source-available, not OSI-open-source: you may read, modify, and self-host the code, but a non-compete clause forbids building a competing commercial product, and each release converts to Apache-2.0 two years after publication. This is the defining tension of the repo — a large, genuinely inspectable codebase that is *not* open source in the OSI sense, and whose maintainers (Sentry Inc.) are also the primary commercial vendor.

The other defining trait is operational weight. Sentry is not a library you drop in; the server is a distributed system with a hard dependency on Kafka, ClickHouse, Redis, PostgreSQL, and several Rust services. Running it yourself is a real infrastructure commitment, which is why most users pay for the hosted product.

## Getting Started

You almost never install this repo to *use* Sentry — you install an SDK and point it at a DSN. Python example:

```bash
pip install sentry-sdk
```

```python
import sentry_sdk

sentry_sdk.init(
    dsn="https://examplePublicKey@o0.ingest.sentry.io/0",
    traces_sample_rate=1.0,      # performance tracing; lower in prod
    send_default_pii=False,
)

def main():
    1 / 0                        # captured, grouped, and reported automatically

sentry_sdk.capture_message("manual breadcrumb")
```

To self-host the *server*, use the companion repo `getsentry/self-hosted` (a docker-compose bundle), not this repo directly:

```bash
git clone https://github.com/getsentry/self-hosted
cd self-hosted && ./install.sh   # brings up ~20 containers
```

Local development of the server itself uses the `devenv` toolchain and `sentry devserver`[^4].

## Architecture / How It Works

The event path is the thing to understand:

1. **Relay** (Rust, separate repo) is the ingestion edge. SDKs POST envelopes to Relay, which authenticates the DSN, applies data-scrubbing / PII rules, enforces rate limits and quotas, and forwards accepted events onto Kafka. Relay exists so the expensive Python backend never sees raw firehose traffic directly.
2. **Kafka** is the ingestion buffer. Consumers (Python) pull events, run grouping (the fingerprinting logic that collapses many events into one issue), symbolication, and normalization.
3. **Snuba** (separate repo) is the query service in front of **ClickHouse**. All the high-cardinality, time-series, search-heavy data (events, spans, sessions, metrics) lives in ClickHouse and is queried through Snuba's abstraction layer, never via raw SQL from the Django app.
4. **PostgreSQL** holds the relational metadata: organizations, projects, users, issue state, alert rules, dashboards.
5. **Symbolicator** (Rust) resolves native/minified stack frames against debug symbols and source maps.
6. The **web app** is a Django backend exposing a REST API, with a large React + TypeScript single-page frontend on top.

So although the top-level language is Python, Sentry is polyglot by necessity: Rust at the latency-critical edges (Relay, Symbolicator), ClickHouse for analytical scale, Python/Django for business logic and orchestration. The coupling is deep — you cannot meaningfully run "just the Django part"; grouping, search, and performance features assume the full pipeline is present.

## Production Notes

- **Self-hosting is heavy.** The `self-hosted` compose stack runs on the order of twenty containers and the maintainers state a minimum of ~16 GB RAM; ClickHouse and Kafka dominate resource use. It is workable for a single team but is not a lightweight appliance, and horizontal scaling beyond the compose defaults is explicitly "you're on your own"[^5].
- **Storage growth is the recurring operational pain.** Event and span volume in ClickHouse and Kafka grows with traffic and with `traces_sample_rate`. Retention windows, sampling, and quota rules are the levers; teams that leave tracing at 100% sampling are the ones that get surprised by disk.
- **Sampling lives in the SDK, not just the server.** `traces_sample_rate` (and dynamic sampling on the server) directly controls both cost and data completeness. Setting it to 1.0 in production is a common footgun for high-throughput services.
- **Upgrades are versioned and occasionally lossy for self-hosters.** The `self-hosted` repo cuts monthly calendar-versioned releases; skipping many versions can require intermediate upgrade steps because of data migrations. Read the release notes before jumping.
- **License compliance is a real design input.** Because FSL forbids offering Sentry as a competing service and older-than-two-years code reverts to Apache-2.0, any internal fork or resale plan needs legal review — this is not MIT/Apache-permissive code.
- **PII / data scrubbing runs at Relay.** Sensitive fields can leak if `send_default_pii=True` or if server-side scrubbing rules are misconfigured; validate scrubbing before pointing production traffic at any Sentry instance, hosted or self-hosted.

## When to Use / When Not

**Use when:**
- You want error tracking with grouped issues, alerting, releases, and distributed tracing across many languages from one backend.
- You need native/minified symbolication (mobile, C/C++, source-mapped JS) handled for you.
- You want the hosted product and treat this repo as reference/transparency rather than something you run.
- Your compliance posture requires self-hosting and you can staff a Kafka/ClickHouse stack.

**Avoid when:**
- You need an OSI-open-source guarantee — FSL is source-available with a non-compete, not open source.
- You want a small, low-dependency self-hosted error logger — the operational footprint is large; a single-binary tool fits better.
- You intend to build a commercial monitoring product on top of it — the license forbids it.
- You only need simple uptime/log aggregation without stack-trace grouping.

## Alternatives

- getsentry/self-hosted — same product; use this when you want to *run* Sentry rather than read/develop the core.
- GlitchTip (glitchtip/glitchtip) — Sentry-SDK-compatible, MIT-licensed, lighter to self-host; use when you want an open-source drop-in for basic error tracking.
- grafana/grafana + OpenTelemetry — use when you want vendor-neutral, OSI-licensed observability and can assemble tracing/metrics/logs yourself.
- highlight/highlight — session replay + error monitoring, Apache-2.0; use when open-source licensing and replay-first workflow matter.
- rollbar / bugsnag (commercial, closed) — use when you want a hosted error tracker and don't want Sentry's tracing/replay surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2008 | Django logging side-project by David Cramer; open-sourced early[^3]. |
| repo created | 2010-08 | `getsentry/sentry` first pushed to GitHub. |
| company | 2012 | Functional Software Inc. (Sentry) founded around the project[^3]. |
| BSL relicense | 2019 | Moves off BSD to the Business Source License[^1]. |
| Performance | 2020 | Distributed tracing / performance monitoring added alongside errors. |
| Session Replay | 2022–2023 | Frontend session replay introduced and rolled out. |
| FSL relicense | 2023-11 | Adopts the Functional Source License (Fair Source); 2-year Apache-2.0 conversion[^1]. |
| Seer (AI) | 2024–2025 | AI-assisted debugging / issue triage surface added. |

## References

[^1]: Sentry, "Relicensing Sentry" and the Functional Source License. https://blog.sentry.io/relicensing-sentry/ and https://fsl.software/
[^2]: Sentry documentation and SDK index. https://docs.sentry.io/
[^3]: David Cramer / Sentry company history and origin as a Django project. https://sentry.io/about/
[^4]: Sentry developer / self-hosted contributing docs (`devenv`, `sentry devserver`). https://develop.sentry.dev/
[^5]: getsentry/self-hosted requirements and monthly release cadence. https://github.com/getsentry/self-hosted

## Tags

python, rust, error-tracking, observability, apm, distributed-tracing, monitoring, django, clickhouse, self-hosted, fair-source, developer-tools
