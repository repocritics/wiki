# grafana/tempo

> Distributed tracing backend that indexes nothing and stores everything in object storage — cheap 100% trace retention, paid for in query latency.

[GitHub repo](https://github.com/grafana/tempo) ·
[Official website](https://grafana.com/oss/tempo/) ·
[License: AGPL-3.0](https://github.com/grafana/tempo/blob/main/LICENSE)

## Overview

Tempo is Grafana Labs' distributed tracing backend, announced at ObservabilityCON in October 2020[^1] and GA as 1.0 in mid-2021. Its founding bet was contrarian: where Jaeger and Zipkin index spans into Cassandra or Elasticsearch (expensive, operationally heavy, usually forcing head sampling), Tempo writes trace data to plain object storage (S3, GCS, Azure Blob, or local disk) with no separate index database. The original version could only retrieve traces by ID; search arrived later and became practical in Tempo 2.0 (February 2023), which switched the block format to Apache Parquet and shipped TraceQL, a PromQL/LogQL-styled query language for traces[^2][^3].

The target user is a team already in the Grafana/Prometheus/Loki orbit that wants to store *all* traces (no sampling) without running a search cluster. The defining tradeoff is cost versus query latency: object storage is cheap enough to keep 100% of spans for weeks, but every search is a columnar scan over Parquet blocks rather than an index lookup, so ad-hoc queries over large tenants take seconds, not milliseconds. Grafana Labs relicensed Tempo (with Grafana and Loki) from Apache-2.0 to AGPL-3.0 in April 2021[^4], which matters if you embed it in a commercial service.

At ~5.4k stars it is the least-starred of the big tracing backends, which understates deployment reality: it ships inside the Grafana LGTM stack and backs Grafana Cloud Traces, so it is operator-installed infrastructure more than a developer-facing library. The repo sees daily pushes as of mid-2026.

## Getting Started

The fastest path is the docker-compose example in the repo (`example/docker-compose`). A minimal single-binary setup:

```yaml
# tempo.yaml — monolithic mode, local storage
server:
  http_listen_port: 3200
distributor:
  receivers:
    otlp:
      protocols:
        grpc:
        http:
storage:
  trace:
    backend: local
    local:
      path: /var/tempo/blocks
    wal:
      path: /var/tempo/wal
```

```bash
docker run -d -v $(pwd)/tempo.yaml:/etc/tempo.yaml \
  -p 3200:3200 -p 4317:4317 \
  grafana/tempo:latest -config.file=/etc/tempo.yaml
```

Point any OTLP exporter (OpenTelemetry SDK or Collector) at `:4317`, then add Tempo as a data source in Grafana (`http://tempo:3200`). Jaeger and Zipkin wire formats are also accepted at the receiver layer.

## Architecture / How It Works

Tempo is a Cortex-lineage horizontally-scalable system built on `grafana/dskit` (hash ring, memberlist gossip) — the same skeleton as Loki and Mimir. It runs either as a single binary (monolithic mode) or as independently scaled microservices[^5]:

- **Distributor** — accepts OTLP, Jaeger, and Zipkin; the receiver layer literally reuses OpenTelemetry Collector receiver code. Hashes trace IDs onto the ingester ring.
- **Ingester** — batches spans into blocks (WAL first), builds bloom filters and trace-ID indexes, flushes completed blocks to object storage.
- **Compactor** — merges small blocks into larger ones and deduplicates spans (with replication factor 3, each span is written three times; compaction reclaims that amplification). Also enforces retention.
- **Query-frontend / Queriers** — the frontend shards a query into many sub-jobs across block ranges; queriers execute them against both ingesters (recent data) and the object storage backend.
- **Metrics-generator** — optional; derives RED-style span metrics and service graphs from the ingest stream and remote-writes them to a Prometheus-compatible TSDB. This is how Tempo powers Grafana's service graph views without a metrics pipeline of its own.

The storage format is the interesting part. Blocks are Parquet files (`vParquet` generations) whose columnar layout is what makes TraceQL feasible: a query touching only `resource.service.name` and `duration` reads only those columns. Frequently queried attributes can be promoted to dedicated columns (vParquet3+) for another large speedup. Because the format is plain Parquet on object storage, blocks are also readable by external tools (DuckDB, Spark) — an escape hatch no index-database competitor offers.

## Production Notes

- **Query latency is the tax.** Trace-by-ID lookup is fast (bloom filters). TraceQL search over a large tenant fans out into thousands of sub-queries scanning Parquet from object storage. Expect seconds; tune query-frontend sharding, run enough queriers, and promote hot attributes to dedicated columns. If your org expects Elasticsearch-style sub-second arbitrary search, Tempo will disappoint.
- **Object storage request costs.** Searches generate GET storms. On S3 this is a real line item and can hit rate limits; a caching layer (memcached is the supported pattern) in front of the backend is effectively mandatory at scale.
- **Metrics-generator cardinality.** Span metrics inherit label dimensions from span attributes. An unbounded attribute (user ID, URL path) will explode series in your Prometheus/Mimir. Configure dimension allowlists before enabling it, not after.
- **Per-tenant limits bite silently.** Ingest over `max_bytes_per_trace`, live-trace, or rate limits results in discarded spans visible only in Tempo's own metrics (`tempo_discarded_spans_total`). Watch that metric from day one; "missing spans" reports almost always trace back to it.
- **Block-format migrations.** Major versions have rotated vParquet generations (vParquet2 → 3 → 4), with older formats deprecated and then dropped. Upgrade sequentially through minor versions and let old blocks age out — skipping several versions can strand unreadable blocks.
- **No built-in sampling.** Tempo assumes you send everything. If you cannot afford full ingest, do tail sampling in an OpenTelemetry Collector in front — Tempo itself offers no sampling controls.
- **AGPL-3.0.** Fine for internal use; a consideration if Tempo is embedded in a product you sell.

## When to Use / When Not

**Use when:**
- You run Grafana already and want traces correlated with Loki logs and Prometheus metrics (trace-to-logs/metrics linking is first-class).
- You want 100% trace retention at object-storage prices instead of sampled retention at Elasticsearch/Cassandra prices.
- Your query pattern is "jump from a log/metric/exemplar to a trace ID" plus occasional TraceQL investigation, not constant full-text search.

**Avoid when:**
- You need low-latency, high-frequency arbitrary search over spans — an indexed or ClickHouse-based backend fits better.
- You are not on the Grafana stack; Tempo's UI story *is* Grafana, and using it standalone forfeits most of its value.
- AGPL is a blocker for your distribution model.

## Alternatives

- jaegertracing/jaeger — use instead for a CNCF-graduated, vendor-neutral backend with pluggable storage (v2 is rebuilt on the OTel Collector) and its own UI.
- openzipkin/zipkin — use instead for a small, mature, batteries-included tracer when you have legacy Zipkin instrumentation.
- SigNoz/signoz — use instead for an all-in-one ClickHouse-backed traces+metrics+logs product with fast columnar search out of the box.
- uptrace/uptrace — use instead for a lighter ClickHouse-based OTel backend with a bundled UI.
- apache/skywalking — use instead for an APM-flavored system popular in the Java ecosystem with built-in agents and topology views.

## History

| Version | Date | Notes |
|---------|------|-------|
| Announce | 2020-10 | Public announcement at ObservabilityCON[^1]. |
| — | 2021-04 | Relicensed Apache-2.0 → AGPL-3.0 (with Grafana, Loki)[^4]. |
| 1.0 | 2021-06 | GA. Trace-by-ID only; no search. |
| 2.0 | 2023-02 | Apache Parquet default block format; TraceQL[^2]. |
| 2.3 | 2023-10 | vParquet3; dedicated attribute columns. |
| 2.4 | 2024-02 | TraceQL metrics (experimental); multi-tenant queries. |
| 2.5 | 2024-05 | vParquet4 default. |
| 2.x | 2024–2026 | Roughly two–three minor releases/year; actively maintained[^6]. |

## References

[^1]: Grafana Labs, "Announcing Grafana Tempo, a massively scalable distributed tracing system" — 2020-10-27. https://grafana.com/blog/2020/10/27/announcing-grafana-tempo-a-massively-scalable-distributed-tracing-system/
[^2]: Grafana Labs, "New in Grafana Tempo 2.0: Apache Parquet as the default storage format, support for TraceQL" — 2023-02-01. https://grafana.com/blog/2023/02/01/new-in-grafana-tempo-2.0-apache-parquet-as-the-default-storage-format-support-for-traceql/
[^3]: TraceQL documentation. https://grafana.com/docs/tempo/latest/traceql/
[^4]: Grafana Labs, "Grafana, Loki, and Tempo will be relicensed to AGPLv3" — 2021-04-20. https://grafana.com/blog/2021/04/20/grafana-loki-tempo-relicensing-to-agplv3/
[^5]: Tempo architecture documentation. https://grafana.com/docs/tempo/latest/operations/architecture/
[^6]: Tempo releases. https://github.com/grafana/tempo/releases

## Tags

go, distributed-tracing, observability, opentelemetry, traceql, object-storage, grafana, parquet, monitoring
