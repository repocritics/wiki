# uptrace/uptrace

> Self-hosted, OpenTelemetry-native APM that stores traces, metrics, and logs in ClickHouse — one UI over all three signals.

[GitHub repo](https://github.com/uptrace/uptrace) ·
[Official website](https://uptrace.dev) ·
[License: AGPL-3.0](https://github.com/uptrace/uptrace/blob/master/LICENSE)

## Overview

Uptrace is an open-source application performance monitoring (APM) tool built around OpenTelemetry as the sole ingestion format and ClickHouse as the storage engine[^1]. It presents traces, metrics, and logs in a single UI — a span waterfall, a PromQL-style metrics explorer, and a log/event search that all share the same faceted-filter model. The project is written in Go (backend) with a Vue frontend, and has been developed by the Uptrace team since late 2021.

The central design bet is columnar storage: rather than the inverted-index approach of Elasticsearch/Jaeger or the trace-store approach of Tempo, Uptrace writes every span, metric point, and log line into ClickHouse tables and answers queries with SQL aggregation. This is what lets a single server absorb high span volume with strong on-disk compression (the README cites ~1KB spans compressing toward ~40 bytes[^1]) at the cost of pulling in a heavyweight analytical database as a hard dependency. Metadata — projects, users, alert definitions, dashboards — lives separately in PostgreSQL.

The repo doubles as the reference deployment for a commercial product: Uptrace also sells a hosted SaaS and an enterprise build. The open-source server is genuinely usable standalone, but some polish and scale-out features are steered by that commercial context, and the license (AGPL-3.0) reflects it — network use triggers source-availability obligations, which matters if you intend to fork and offer it as a service.

## Getting Started

The supported quickstart is Docker Compose, which brings up Uptrace, ClickHouse, and PostgreSQL together[^2]:

```bash
git clone https://github.com/uptrace/uptrace.git
cd uptrace/example/docker
docker compose up -d
# UI on http://localhost:14318 ; OTLP ingest on ports 14317 (gRPC) / 14318 (HTTP)
```

Point any OpenTelemetry SDK or Collector at the OTLP endpoint using the project's DSN. Example with the Go SDK exporter distributed by the same org:

```go
import "github.com/uptrace/uptrace-go/uptrace"

uptrace.ConfigureOpentelemetry(
    uptrace.WithDSN("http://project2_secret@localhost:14318?grpc=14317"),
    uptrace.WithServiceName("myservice"),
    uptrace.WithServiceVersion("1.0.0"),
)
defer uptrace.Shutdown(ctx)
```

Any OTLP-compliant emitter works — you are not required to use Uptrace's own SDK. Projects, users, and DSNs are declared in a YAML config file (`uptrace.yml`) rather than a settings UI.

## Architecture / How It Works

Three moving parts, and you own all of them when self-hosting:

1. **Uptrace server (Go)** — terminates OTLP (gRPC + HTTP), batches writes into ClickHouse, evaluates alerting rules, and serves the API and Vue SPA. Also accepts Prometheus, Vector, FluentBit, and CloudWatch inputs, normalizing them into OTel signals[^1].
2. **ClickHouse** — the hot path. Spans, metric samples, and log records each land in their own tables. Queries are SQL aggregations; the span query language and metric query language compile down to ClickHouse SQL. Retention is enforced with ClickHouse TTL, not by an application-side reaper.
3. **PostgreSQL** — metadata store: dashboards, monitors/alerts, users, saved views. Small and transactional; not on the ingestion hot path.

The signal correlation that makes the UI feel unified is mostly a query-time join over shared attributes (trace IDs, service names, timestamps) rather than a single physical store — the three tables stay separate but share OpenTelemetry's semantic-convention attribute keys, which is what lets a facet like `service.name` filter traces, logs, and metrics identically.

Because storage is ClickHouse, cardinality behaves differently than in a Prometheus-style TSDB: high-cardinality attributes are cheaper to filter on (they are just columns) but the tradeoff moves to ClickHouse's own merge/compaction and memory characteristics under heavy aggregation. The scaling story is "make ClickHouse bigger / clustered," not "shard the app."

## Production Notes

- **You are running ClickHouse in production.** This is the real operational cost. ClickHouse tuning (MergeTree settings, TTL, `max_memory_usage`, disk provisioning, backups) becomes your problem. Teams without prior ClickHouse experience should budget for that learning curve; a mis-sized single node is the most common failure mode, not the Uptrace binary itself.
- **Single-server framing has limits.** The "billions of spans on one server" claim is real for ingest throughput given fast disks, but query latency under wide time-range aggregations, retention pressure, and concurrent users all push you toward a larger or clustered ClickHouse well before the Go server is the bottleneck.
- **Two databases to back up, on different cadences.** ClickHouse holds the (large, TTL-expiring) telemetry; PostgreSQL holds the (small, must-not-lose) config and alert state. A backup plan that covers only one leaves you exposed.
- **AGPL-3.0.** Fine for internal self-hosting. If you embed Uptrace in a product you offer over a network, the copyleft obligations attach. Evaluate before building a managed offering on top of it.
- **Config is file-driven.** Users, projects, and DSNs live in YAML. This is good for GitOps and bad if you expected multi-tenant self-service signup out of the box.
- **Sampling still matters.** Uptrace ingests what you send; controlling cost/volume is done upstream in the OpenTelemetry Collector (tail/head sampling), not primarily inside Uptrace.
- **Open-source vs. hosted feature gap.** Some capabilities and scale features are steered by the commercial editions. Confirm a given feature is in the OSS server before designing around it rather than assuming parity with the docs, which describe the product broadly.

## When to Use / When Not

**Use when:**
- You are already all-in on OpenTelemetry and want one self-hosted UI for traces, metrics, and logs instead of stitching Jaeger + Prometheus + Loki.
- You want SQL-grade slicing over high-cardinality trace/log data and are comfortable operating ClickHouse.
- Cost control via self-hosting matters and you have the ops capacity to run analytical infra.

**Avoid when:**
- You don't want to operate ClickHouse — a managed vendor (Datadog, Grafana Cloud, Honeycomb) or a lighter self-hosted stack will cost less operational attention.
- Your telemetry is metrics-first and Prometheus + Grafana already satisfies you; adding ClickHouse buys little.
- The AGPL license conflicts with how you intend to redistribute or offer the software.
- You need a large third-party plugin/integration ecosystem — Grafana's is far deeper.

## Alternatives

- grafana/grafana + tempo + loki + prometheus — the modular self-hosted stack; more components to run, larger ecosystem, no single unified query model.
- jaegertracing/jaeger — tracing only; pair with something else for metrics/logs when you don't want a combined store.
- SigNoz (signoz/signoz) — the closest peer: also OpenTelemetry-native and ClickHouse-backed, three-signals-in-one-UI; evaluate both directly.
- openobserve/openobserve — self-hosted observability with its own columnar/object-store engine; use when you want cheaper object-storage retention over ClickHouse ops.
- Datadog / Honeycomb (SaaS) — use when you would rather pay to not run any of this infrastructure.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2021-12 | Repo opened; OpenTelemetry + ClickHouse APM[^3]. |
| v1.x | 2022–2023 | Unified traces/metrics/logs UI, alerting, SSO via OIDC. |
| ongoing | 2026-06 | Actively maintained; last push 2026-06-14 on `master`[^3]. |

Version-level dates beyond the repo's creation and last-push timestamps are not asserted here to avoid citing release numbers that could not be verified against the live repo at authoring time.

## References

[^1]: uptrace/uptrace README — feature list, ingestion sources, compression and throughput claims. https://github.com/uptrace/uptrace
[^2]: Docker Compose example deployment. https://github.com/uptrace/uptrace/tree/master/example/docker
[^3]: GitHub repository metadata (created 2021-12-22, last push 2026-06-14, default branch `master`, license AGPL-3.0), retrieved via GitHub API 2026-07. https://github.com/uptrace/uptrace

## Tags

go, observability, apm, opentelemetry, distributed-tracing, metrics, logs, clickhouse, self-hosted, monitoring, telemetry
