# SigNoz/signoz

> OpenTelemetry-native observability (traces, metrics, logs) on a ClickHouse backend — an open-core, self-hostable alternative to Datadog/New Relic.

[GitHub repo](https://github.com/SigNoz/signoz) ·
[Official website](https://signoz.io) ·
License: MIT Expat (core) / SigNoz Enterprise (`ee/`)[^license]

## Overview

SigNoz is a single-application observability platform: traces, metrics, and logs land in one store and share one query and dashboard surface, so you can pivot from a latency chart to the underlying spans to the correlated logs without changing tools[^readme]. It was founded in 2020, went through Y Combinator (W21), and the repo opened in January 2021[^created]. The pitch is an open-source competitor to Datadog / New Relic / Grafana-stack fragmentation, built entirely on OpenTelemetry as the ingestion contract.

The defining architectural bet is **ClickHouse as the single datastore**. Every signal — spans, metric samples, log lines — is written to ClickHouse tables and queried through SigNoz's own query builder (which also exposes PromQL and raw ClickHouse SQL)[^readme]. That choice is the source of both its strengths (columnar storage handles high-cardinality, high-volume telemetry cheaply; one database to operate instead of Prometheus + a tracing store + a log store) and its operational reality (you now run and scale ClickHouse, which is the real production burden).

The project is **open-core**, not fully permissive. Code outside `ee/` and `cmd/enterprise/` is MIT Expat; those directories are under a separate SigNoz Enterprise license that restricts production use to paying customers[^license]. SSO, granular RBAC, and some ingestion/retention controls live behind that line. As of mid-2026 the project is still pre-1.0 (latest release v0.132.2, 2026-07-10)[^release] despite ~29.5k stars and heavy daily commit activity[^repo] — the version number understates maturity but does signal a fast, occasionally-breaking release cadence.

## Getting Started

The fastest self-hosted path is Docker Compose for evaluation:

```bash
git clone -b main https://github.com/SigNoz/signoz.git
cd signoz/deploy
./install.sh          # brings up ClickHouse, the OTel collector, query-service, and the UI
# UI: http://localhost:8080
```

Then point an application at the bundled OpenTelemetry Collector (default OTLP gRPC on `4317`):

```bash
# Example: a Node service exporting OTLP to a local SigNoz collector
OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317" \
OTEL_RESOURCE_ATTRIBUTES="service.name=checkout" \
OTEL_TRACES_EXPORTER=otlp \
node --require @opentelemetry/auto-instrumentations-node/register server.js
```

Because ingestion is plain OTLP, any OpenTelemetry SDK or the upstream Collector can send to SigNoz — you are not locked into a proprietary agent.

## Architecture / How It Works

SigNoz is several processes, not a monolith:

1. **OpenTelemetry Collector** — SigNoz ships `signoz-otel-collector`, a fork/distribution of the upstream Collector with a ClickHouse exporter and SigNoz-specific processors. It receives OTLP and writes to ClickHouse. You can also run a vanilla upstream Collector with the ClickHouse exporter configured to the same schema.
2. **ClickHouse** — the storage and query engine. Traces, logs, and metrics each have their own table families. Retention is enforced with ClickHouse TTLs; tiered (hot/cold) storage to S3-compatible object stores is supported through ClickHouse's own storage policies.
3. **query-service** — a Go backend that translates the visual query builder, PromQL, and dashboard/alert definitions into ClickHouse SQL, and evaluates alert rules.
4. **Frontend** — a React/TypeScript single-page app for the explorer, dashboards, service map, and trace flamegraphs.

The query builder is the connective tissue: a metric panel, a trace filter, and a log search all compile down to ClickHouse SQL over a shared schema, which is what makes cross-signal correlation (trace ID → logs, exemplar → span) work without a separate join layer.

Early SigNoz architecture experimented with an Apache Druid + Kafka pipeline before standardizing on ClickHouse; the ClickHouse-native design is what shipped and stabilized[^clickhouse]. Metrics ingestion is OTLP-first but Prometheus remote-write and PromQL are supported so existing Prometheus users can migrate incrementally.

## Production Notes

**You are operating ClickHouse, whether you meant to or not.** The Docker Compose default runs a single-node ClickHouse that is fine for evaluation and small workloads. Production scale means a ClickHouse cluster with ClickHouse Keeper (or ZooKeeper), replication, and sharding — and the operational knowledge that entails (background merges, part explosions, `max_memory_usage`, disk pressure). This is the single biggest gap between "it works on my laptop" and "it survives our traffic."

**Compose → Kubernetes is a real migration, not a flag.** Most serious deployments move to the Helm chart (`signoz/charts`) on Kubernetes. Data does not automatically follow you from a Compose install; plan the cutover.

**Schema migrations across upgrades.** The pre-1.0 cadence means schema and pipeline changes ship in minor releases. Trace and log schemas have been revised across versions; upgrades sometimes run migration jobs, and skipping several releases at once is riskier than incremental upgrades. Read release notes before jumping versions.

**Cardinality is cheaper here, not free.** ClickHouse tolerates high-cardinality attributes far better than Prometheus/Loki, but query cost still tracks scanned data. Materialized columns and sensible attribute indexing matter for dashboards over large log volumes.

**Retention and cost live in ClickHouse config.** There is no magic tiering UI for the open-source edition beyond what you configure — retention is TTLs, cold storage is a ClickHouse S3 storage policy. Get this right early; changing TTLs on populated tables triggers expensive rewrites.

**Open-core boundaries.** SSO (SAML/OIDC), fine-grained RBAC, and some enterprise ingestion/retention controls are gated behind the `ee/` license[^license]. If your compliance story depends on SSO + RBAC, budget for SigNoz Cloud or an enterprise contract rather than assuming the OSS build covers it.

## When to Use / When Not

**Use when:**
- You want traces + metrics + logs correlated in one self-hosted tool and are willing to own ClickHouse.
- You are already OpenTelemetry-instrumented (or migrating to it) and want a vendor-neutral backend.
- You want off Datadog/New Relic per-host or per-seat pricing and can trade dollars for operational effort.
- High-cardinality logs/traces make Prometheus + Loki painful.

**Avoid when:**
- You only need metrics — Prometheus alone is simpler and lighter.
- You have no appetite for running ClickHouse and want a fully managed experience (use SigNoz Cloud or a commercial SaaS instead).
- You need SSO/RBAC and refuse a paid tier — those are enterprise-licensed.
- You need a battle-tested 1.0 with strict long-term schema stability guarantees; the fast 0.x cadence is a poor fit.

## Alternatives

- grafana/grafana + prometheus/prometheus + Loki + Tempo — assemble-your-own stack; more flexible and more pieces to run. Use it when you want best-of-breed components over a single integrated store.
- jaegertracing/jaeger — use when you only need distributed tracing and already have separate metrics/logs.
- open-telemetry/opentelemetry-collector — the ingestion layer, not a backend; pair it with any store. Use when you want to keep the pipeline and swap backends.
- uptrace/uptrace — the closest architectural twin (OpenTelemetry + ClickHouse observability). Use when you want a similar design with a different feature/UX tradeoff.
- elastic/elasticsearch (with APM) — use when you are already an Elastic shop, accepting higher ingestion resource cost for logs.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2021-01 | Repo opened; project public[^created]. |
| — | W21 | Y Combinator batch; storage standardized on ClickHouse[^clickhouse]. |
| 0.x | ongoing | Frequent releases; traces/logs schema revised across versions[^release]. |
| v0.132.2 | 2026-07-10 | Latest release; project remains pre-1.0[^release]. |

## References

[^repo]: GitHub API `repos/SigNoz/signoz` — ~29,532 stars, 2,325 forks, ~1,517 open issues, last push 2026-07-14. Fetched 2026-07-15. https://github.com/SigNoz/signoz
[^license]: SigNoz `LICENSE` — MIT Expat for content outside `ee/` and `cmd/enterprise/`; those directories under the SigNoz Enterprise license (`ee/LICENSE`). GitHub reports NOASSERTION because of the split. https://github.com/SigNoz/signoz/blob/main/LICENSE
[^readme]: SigNoz README — OpenTelemetry-native, single columnar (ClickHouse) datastore, Query Builder / PromQL / ClickHouse SQL, correlated signals. https://github.com/SigNoz/signoz
[^created]: Repository `created_at` = 2021-01-03 (GitHub API). https://github.com/SigNoz/signoz
[^release]: Latest release v0.132.2, published 2026-07-10 (GitHub API `releases/latest`). https://github.com/SigNoz/signoz/releases
[^clickhouse]: SigNoz architecture documentation — ClickHouse as the datastore for traces, metrics, and logs. https://signoz.io/docs/architecture/

## Tags

observability, opentelemetry, apm, distributed-tracing, log-management, metrics, clickhouse, monitoring, self-hosted, golang, typescript, open-core
