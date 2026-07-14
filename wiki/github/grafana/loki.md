# grafana/loki

> Log aggregation that indexes labels, not content — "like Prometheus, but for logs."

[GitHub repo](https://github.com/grafana/loki) ·
[Official website](https://grafana.com/oss/loki) ·
[License: AGPL-3.0](https://github.com/grafana/loki/blob/main/LICENSE)

## Overview

Loki is a horizontally-scalable, multi-tenant log aggregation system from Grafana Labs, first shown at KubeCon in December 2018 and reaching 1.0 in November 2019[^1]. Its defining design choice is that it does **not** build a full-text index over log content. Instead it indexes only a small set of labels per log stream (the same label model Prometheus uses for metrics) and stores the raw log lines as compressed chunks in object storage. The bet is that most of the cost and operational pain of systems like Elasticsearch comes from inverted-index maintenance, and that you can skip it if you're willing to brute-force-scan content at query time.

That bet is the central tension of operating Loki. Ingest is cheap and storage is cheap (compressed chunks on S3/GCS/Azure), but a query that filters on log *content* rather than labels has to fetch and decompress every chunk in the matching streams over the requested time range. Loki is fast when your label selector narrows the search to a few streams and slow-and-expensive when it doesn't — so label design is not a nicety, it's the thing that determines whether the system works for you.

Loki is squarely aimed at cloud-native, Kubernetes-centric shops already running Prometheus and Grafana. The stack is three parts: an agent (Grafana Alloy, which replaced Promtail[^2]) to collect and push logs, Loki to store and query, and Grafana to visualize. It is tightly coupled to that ecosystem — LogQL mirrors PromQL, the query UI lives in Grafana, and the managed offering is Grafana Cloud Logs.

## Getting Started

Single-binary local run from source (needs a recent Go):

```bash
git clone https://github.com/grafana/loki
cd loki
go build ./cmd/loki
./loki -config.file=./cmd/loki/loki-local-config.yaml
```

A LogQL query — stream selector by labels, then a content filter, aggregated to a rate:

```logql
# per-second rate of 5xx lines from the api app, last selector is label-indexed,
# the |= and | json steps brute-force-scan matched chunks
sum by (route) (
  rate(
    {app="api", namespace="prod"} |= "status=5" | json | status >= 500 [5m]
  )
)
```

Push logs directly over the HTTP API (normally the agent's job):

```bash
curl -H "Content-Type: application/json" -XPOST \
  "http://localhost:3100/loki/api/v1/push" --data-raw \
  '{"streams":[{"stream":{"app":"api","level":"error"},
    "values":[["'$(date +%s)000000000'","boom: db timeout"]]}]}'
```

## Architecture / How It Works

Loki is a set of components that can run as one process or be split apart. It offers three deployment shapes:

1. **Monolithic** — every component in a single binary; fine up to modest volumes.
2. **Simple scalable** (read / write / backend targets) — the recommended middle ground for most production installs.
3. **Microservices** — each component scaled independently; maximum flexibility, maximum operational cost.

The core components on the **write path**: the *distributor* validates incoming streams and, using a consistent-hash ring (coordinated via a memberlist gossip ring), routes each stream to *ingesters* (typically replication factor 3). Ingesters accumulate log lines into in-memory chunks, back them with a write-ahead log, and periodically flush compressed chunks to object storage. The **read path**: a *query-frontend* splits and sub-shards queries and manages a results cache, handing work to a *query-scheduler* and *queriers*; queriers stitch together recent data from ingesters with older chunks from object storage.

**Storage** is two things: chunks (the compressed logs) and an index (label → chunk pointers). The index engine has evolved: early Loki used external NoSQL stores (Cassandra, BigTable, DynamoDB), then `boltdb-shipper` shipped a self-contained index to object storage, and the current default is a **TSDB index** borrowed from Prometheus[^3]. As of 3.0 the legacy chunk/index stores are deprecated and object storage (S3, GCS, Azure Blob, filesystem) with the TSDB index and schema v13 is the supported path.

**LogQL** is the query language: a Prometheus-style label selector `{app="api"}`, line filters (`|=`, `!=`, `|~`, `!~`), parsers (`json`, `logfmt`, `pattern`, `regexp`), label/line formatting, and metric queries (`rate`, `count_over_time`, `sum by`) that turn logs into time series. **Structured metadata** (3.0) lets you attach high-cardinality fields like trace IDs to a line without making them index labels[^4]. **Bloom filters** were introduced as an experimental accelerator for needle-in-haystack content searches and remain a moving target — treat them as not-yet-stable.

Multi-tenancy runs throughout via the `X-Scope-OrgID` header; every tenant has isolated streams, limits, and retention.

## Production Notes

**Cardinality is the footgun that eats clusters.** Every unique combination of label values is a separate stream, and streams are the unit of index and ingester memory. Putting a high-cardinality field (user ID, pod IP, trace ID, request path with IDs) into a label multiplies streams into the millions and takes ingesters down. The discipline: keep labels few and low-cardinality (app, namespace, level, cluster), and push everything else into the log line or into structured metadata, retrieved with filter expressions at query time.

**"No index" means content queries scan.** Because content isn't indexed, a query whose label selector is broad has to decompress large volumes of chunks. Loki performance is really query-frontend sharding + parallelism + caching (results cache, chunks cache, index cache). Under-provisioned caches and a missing query-scheduler are common reasons a cluster "feels slow." Long time ranges over broad selectors are the expensive case; narrow the streams first.

**License.** Loki was relicensed from Apache-2.0 to **AGPL-3.0** in April 2021 alongside Grafana and Tempo[^5]. For most operators self-hosting internally this is a non-issue, but teams that embed or offer Loki as part of a service to third parties need to read the AGPL implications; Grafana Enterprise Logs (GEL) is the proprietary commercial variant.

**Upgrades are not free.** The 2.x → 3.0 jump removed deprecated index stores and changed many config defaults, requiring a schema/period-config migration to TSDB + schema v13 and, in some setups, object-storage client (Thanos-based) config changes[^6]. Schema changes are additive via dated `schema_config` periods — you add a new period rather than rewriting old data — but planning the cutover date and validating the new index is mandatory. Read the version-specific upgrade guide before every minor bump.

**Other operator realities:** out-of-order writes per stream are accepted within a window (default behavior since 2.4), but very late data still gets rejected; retention and deletes run through the *compactor* and its deletion API rather than object-lifecycle rules alone; the ingester WAL matters for surviving restarts without chunk loss; and per-tenant limits (max streams, ingestion rate, max query length) are the main defense against a noisy tenant.

## When to Use / When Not

**Use when:**
- You already run Prometheus + Grafana and want logs correlated with metrics under the same label model.
- Log volume is high and cost matters more than rich full-text/relevance search.
- You're collecting Kubernetes pod logs where labels come for free from pod metadata.
- Your queries mostly start from a known service/namespace and filter down.

**Avoid when:**
- You need full-text relevance search, fuzzy matching, or ad-hoc analytics across all content — an inverted-index engine fits better.
- Your access pattern is "search all logs for this string with no idea which service" over long ranges.
- You can't commit to disciplined low-cardinality labeling.
- You want a turnkey single-node logging appliance with minimal tuning — Loki rewards operators who understand its caching and sharding.

## Alternatives

- elastic/elasticsearch — full-text inverted index; far richer search and analytics, much heavier to run and store. Use when relevance search and arbitrary content queries are the point.
- opensearch-project/OpenSearch — Apache-2.0 Elasticsearch fork; same tradeoff, different license/governance.
- VictoriaMetrics/VictoriaLogs — cost-efficient, low-cardinality-friendly logs with a similar economics story; use when you want Loki-like efficiency outside the Grafana stack.
- quickwit-oss/quickwit — full-text search over object storage; use when you want cheap storage *and* real full-text indexing.
- ClickHouse/ClickHouse — columnar SQL store often used as a log backend; use when you want SQL analytics and can model schemas yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| announce | 2018-12 | Unveiled at KubeCon Seattle[^1]. |
| 1.0 | 2019-11 | First GA release. |
| 2.0 | 2020-10 | LogQL enhancements, `boltdb-shipper` self-contained index. |
| 2.4 | 2021-11 | Out-of-order ingestion accepted by default; simplified deployment. |
| 2.8 | 2023-04 | TSDB index matures toward default. |
| 2.9 | 2023-09 | Last of the 2.x line before the 3.0 breaks. |
| 3.0 | 2024-04 | Structured metadata, experimental bloom filters, TSDB default, native OTel ingest, legacy store removal[^4]. |

## References

[^1]: Grafana Labs, "Loki: Prometheus-inspired, open source logging for cloud natives" — 2018-12. https://grafana.com/blog/2018/12/12/loki-prometheus-inspired-open-source-logging-for-cloud-natives/
[^2]: Loki README — Alloy replaced Promtail as the recommended collection agent. https://github.com/grafana/loki
[^3]: Loki storage / TSDB index documentation. https://grafana.com/docs/loki/latest/operations/storage/tsdb/
[^4]: Grafana Labs, "Loki 3.0 release" — 2024-04. https://grafana.com/blog/2024/04/09/grafana-loki-3.0-release/
[^5]: Grafana Labs, "Grafana, Loki, and Tempo will be relicensed to AGPLv3" — 2021-04-20. https://grafana.com/blog/2021/04/20/grafana-loki-tempo-relicensing-to-agplv3/
[^6]: Loki upgrade guide. https://grafana.com/docs/loki/latest/setup/upgrade/

## Tags

go, logging, observability, log-aggregation, cloud-native, kubernetes, prometheus, grafana, time-series, distributed-systems, agpl
