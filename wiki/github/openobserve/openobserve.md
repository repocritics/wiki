# openobserve/openobserve

> Single-binary, S3-native observability backend (logs, metrics, traces, RUM) built on Rust and Apache DataFusion, positioned as a self-hostable Datadog/Elasticsearch alternative.

[GitHub repo](https://github.com/openobserve/openobserve) ·
[Official website](https://openobserve.ai) ·
[License: AGPL-3.0](https://github.com/openobserve/openobserve/blob/main/LICENSE)

## Overview

OpenObserve (O2) is an observability platform that ingests logs, metrics, traces, and Real User Monitoring (RUM) data and stores it as Parquet on object storage (S3, GCS, Azure Blob, MinIO, or local disk). The query engine is Apache Arrow DataFusion, so queries are written in SQL for logs/traces and SQL or PromQL for metrics — there is no proprietary query language to learn. Despite GitHub classifying the repo as TypeScript (that count reflects the Vue web UI under `web/`), the core engine is Rust and ships as a single binary.[^1]

The project began at Zinc Labs (founder Prabhat Sharma), which first shipped Zinc, a lightweight full-text search server, then a Rust rewrite aimed at observability called ZincObserve, later renamed OpenObserve.[^2] The defining bet is columnar-on-object-storage: rather than keeping data on hot SSD in an inverted index like Elasticsearch, O2 writes Parquet files to cheap object storage and relies on partitioning, file-level statistics, and a caching layer to keep queries fast. This is where the marketed "140x lower storage cost" comes from — a genuine architectural difference, but the specific multiplier is a vendor benchmark against a particular Elasticsearch configuration and should be read as "much cheaper storage," not a guaranteed number for your workload.[^3]

The other defining tension is open-core. The AGPL-3.0 edition is genuinely feature-complete for ingest, search, dashboards, alerts, and pipelines, but several capabilities most enterprises consider table-stakes — RBAC, SSO, Sensitive Data Redaction, audit trails, federated/Super Cluster search — live only in the commercial Enterprise edition.[^4]

## Getting Started

Single-node via Docker (data persisted to a local volume):

```bash
docker run -d \
  --name openobserve \
  -v $PWD/data:/data \
  -p 5080:5080 \
  -e ZO_ROOT_USER_EMAIL="root@example.com" \
  -e ZO_ROOT_USER_PASSWORD="Complexpass#123" \
  public.ecr.aws/zinclabs/openobserve:latest
```

The UI and API are on port 5080. Point an OpenTelemetry Collector (or any OTLP exporter) at the OTLP HTTP endpoint, or use the Elasticsearch-compatible `_bulk` ingest API to migrate existing shippers:

```bash
curl -u root@example.com:Complexpass#123 \
  -X POST http://localhost:5080/api/default/default/_json \
  -H "Content-Type: application/json" \
  -d '[{"level":"info","message":"hello","service":"demo"}]'
```

By default a single node keeps metadata in an embedded SQLite database and stores Parquet on local disk; point `ZO_S3_*` / storage env vars at object storage for durable deployments.

## Architecture / How It Works

A single OpenObserve binary internally plays several roles — router, ingester, querier, compactor, and (in clustered mode) a coordination node. In single-node mode all roles run in one process; in High Availability mode you run them as separate deployments and scale each independently.[^5]

The data path:

1. **Ingest** — data arrives via OTLP, the Elasticsearch bulk API, or native JSON. Records are grouped into streams (the O2 equivalent of an index/table) scoped to an organization.
2. **Buffer + write** — recent data is buffered, then flushed to columnar **Parquet** files. Compaction later merges small files into larger, better-partitioned ones.
3. **Object storage** — Parquet lands on S3/GCS/Azure/MinIO/local. This is the durable tier; nodes are stateless with respect to it.
4. **Query** — **Apache DataFusion** plans and executes SQL over the Parquet files. Partition pruning and file-level statistics skip files that cannot match; a local disk/memory cache holds hot files to avoid repeated object-storage round trips.

**Metadata and coordination.** A single node uses SQLite. HA deployments use PostgreSQL or MySQL for metadata plus **etcd or NATS** for cluster coordination and eventing. Getting this tier right (and backed up) is the real operational work in a clustered install — the Parquet in object storage is durable, but the metadata store is the source of truth for stream schemas, users, and dashboards.

**Full-text search** is served by an inverted index built alongside the Parquet partitions (rather than scanning raw columns), which keeps log-search latency reasonable without an Elasticsearch-style always-hot index.

The consequence of the columnar-on-object-store design is that OpenObserve is excellent at analytical aggregations over large volumes and at cheap long retention, and comparatively more dependent on its cache tier for the low-latency, needle-in-haystack lookups that a hot inverted index handles natively.

## Production Notes

**Data is immutable.** Once ingested, records cannot be updated or individually deleted — you can only drop whole retention windows.[^6] This is fine (arguably desirable) for logs/traces/audit, but rules out GDPR-style per-record deletion and any workflow that expects mutation.

**Object-storage latency is real.** Queries that hit uncached Parquet pay S3 request + transfer latency. Sizing the disk/memory cache to your hot working set is the single biggest performance lever; a cold query over cold storage is not comparable to a hot Elasticsearch query. Budget cache capacity, not just compute.

**HA is a different animal than single-binary.** The "single binary in 2 minutes" story is genuine for evaluation and small deployments, but production HA means running separate router/ingester/querier/compactor tiers plus an external Postgres/MySQL and etcd/NATS. Plan capacity and backups for the metadata layer accordingly.

**Open-core gates matter.** RBAC, SSO (OIDC/SAML/LDAP), Sensitive Data Redaction, audit trails, and multi-cluster federated search are Enterprise-only. If your security review requires SSO + granular RBAC, the AGPL edition alone will not satisfy it — factor the commercial license into any serious rollout.[^4]

**AGPL-3.0.** The network-copyleft license is unproblematic for internal use but has implications if you offer a modified OpenObserve as a service to third parties. Read it before embedding or reselling.

**Vendor benchmarks.** The "140x storage" and "1/4 the hardware / faster than Elasticsearch" figures come from the maintainer, not an independent study. The direction (columnar + object storage is cheaper) is credible; the exact magnitude depends heavily on data shape, cardinality, and retention. Benchmark your own workload before committing.

**PromQL parity.** Metrics support PromQL, but treat it as a compatible implementation rather than Prometheus itself — validate the specific functions and recording/alerting-rule behavior your dashboards rely on.

## When to Use / When Not

**Use when:**
- You want one self-hosted tool for logs, metrics, traces, and RUM instead of stitching Loki + Prometheus + Tempo + Grafana.
- Storage cost and long retention dominate your bill and object storage is available.
- You are OpenTelemetry-native and want SQL/PromQL rather than a proprietary query language.
- You are evaluating and want something running in minutes from a single binary.

**Avoid when:**
- You need per-record mutation/deletion (immutability is fundamental).
- SSO + fine-grained RBAC are hard requirements and you cannot buy the Enterprise edition.
- You depend on Elasticsearch's mature full-text/search-DSL ecosystem or Prometheus's exact PromQL/rule semantics.
- Your priority is ultra-low-latency point lookups on cold data without investing in cache sizing.

## Alternatives

- grafana/loki — use when you are already committed to the Grafana + Prometheus stack and want logs to fit that model rather than a unified all-signals platform.
- elastic/elasticsearch — use when full-text search, the query DSL, and the broader ecosystem matter more than storage cost or a single-binary footprint.
- SigNoz/signoz — closest philosophical competitor (OpenTelemetry-native, unified signals) but built on ClickHouse; use it if you prefer a ClickHouse storage engine.
- ClickHouse/ClickHouse — use when you want a general columnar OLAP database and are willing to build the observability layer yourself.
- prometheus/prometheus — use when metrics are the whole job and you want the reference PromQL implementation, not a compatible reimplementation.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2022 | Zinc Labs ships Zinc, a lightweight full-text search server (the precursor project).[^2] |
| — | 2023-02 | This repo created; Rust observability engine (then ZincObserve) begins public development.[^1] |
| — | 2023 | Renamed ZincObserve → OpenObserve; project scope settles on unified logs/metrics/traces.[^2] |
| — | 2023 | License changed from Apache-2.0 to AGPL-3.0 for the open-source edition.[^3] |
| — | 2024–2026 | Pipelines, RUM/frontend monitoring, alerting, and HA/Super Cluster (Enterprise) added; actively developed with frequent releases.[^7] |

## References

[^1]: OpenObserve repository and README. https://github.com/openobserve/openobserve
[^2]: OpenObserve / Zinc Labs background. https://openobserve.ai
[^3]: OpenObserve blog, "Why AGPL and why it's good for the community." https://openobserve.ai/blog/what-are-apache-gpl-and-agpl-licenses-and-why-openobserve-moved-from-apache-to-agpl/
[^4]: OpenObserve enterprise features and downloads/comparison. https://openobserve.ai/downloads/
[^5]: OpenObserve architecture documentation. https://openobserve.ai/docs/architecture/
[^6]: OpenObserve README FAQ — data immutability. https://github.com/openobserve/openobserve#-faq
[^7]: OpenObserve releases. https://github.com/openobserve/openobserve/releases

## Tags

observability, logs, metrics, tracing, opentelemetry, rust, datafusion, parquet, s3, self-hosted, apm, elasticsearch-alternative
