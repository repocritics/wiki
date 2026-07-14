# VictoriaMetrics/VictoriaMetrics

> A Go time series database and Prometheus-compatible monitoring backend, optimized for low RAM use and high compression on high-cardinality metrics.

[GitHub repo](https://github.com/VictoriaMetrics/VictoriaMetrics) ·
[Official website](https://victoriametrics.com/) ·
[License: Apache-2.0](https://github.com/VictoriaMetrics/VictoriaMetrics/blob/master/LICENSE)

## Overview

VictoriaMetrics is a time series database (TSDB) and metrics monitoring system, started by Aliaksandr Valialkin (author of `fasthttp`) with the repository created in 2018[^1]. Its pitch is narrow and consistent: ingest the same data as Prometheus, store it with substantially less RAM and disk, and query it with PromQL — plus a superset dialect called MetricsQL. As of 2026 it is one of the most-adopted Prometheus long-term-storage backends, at ~17.3k stars and still pushed to daily[^2], with production references including Grammarly, Roblox, Wix, and CERN-scale telemetry.

The project ships in two open-source forms under Apache-2.0: a **single-node** binary and a **cluster** version. Both are genuinely open source — this is not an open-core-only story — but a separately licensed **Enterprise** build gates several features operators eventually want at scale: downsampling, multiple retention periods per dataset, automated backups, and `vmanomaly` anomaly detection[^3]. The defining tension is exactly there: the OSS core is fast and cheap to run, but the features that most reduce long-term storage cost (downsampling, per-tenant retention) sit behind the commercial line.

The second tension is dialect. MetricsQL is a strict-ish superset of PromQL, but its default rollup and extrapolation semantics differ from Prometheus in subtle ways, so a dashboard that looks identical can return slightly different numbers than a native Prometheus source[^4].

## Getting Started

```bash
# Single-node: one static binary, no external dependencies.
docker run -it --rm -p 8428:8428 \
  victoriametrics/victoria-metrics \
  -retentionPeriod=3   # months; default is 1
```

Point Prometheus at it via `remote_write`, then query with PromQL/MetricsQL on port 8428:

```yaml
# prometheus.yml
remote_write:
  - url: http://victoriametrics:8428/api/v1/write
```

```bash
# Instant query (PromQL-compatible HTTP API).
curl 'http://localhost:8428/api/v1/query' \
  --data-urlencode 'query=sum(rate(http_requests_total[5m])) by (job)'
```

VictoriaMetrics can also scrape targets directly (a subset of Prometheus scrape config) or ingest via InfluxDB line protocol, Graphite, OpenTSDB, CSV, and OpenTelemetry — so it can replace Prometheus outright rather than only sit behind it.

## Architecture / How It Works

The **single-node** binary is one process handling ingestion, storage, and query. Storage is a custom LSM-like engine (`mergeset`) that separates the inverted index (label → time series) from the columnar data blocks, and compresses timestamps and values with delta/Gorilla-style encoding. The high-compression and low-RAM claims come from this design plus aggressive background merges; the tradeoff is that heavy backfilling of old timestamps forces large merge storms.

The **cluster** version splits into three stateless-or-stateful roles[^5]:

- **`vminsert`** — stateless; accepts writes, shards each series across storage nodes by label-set hash.
- **`vmstorage`** — stateful; owns the data on local disk. Nodes do **not** talk to each other.
- **`vmselect`** — stateless; fans a query out to all storage nodes and merges results.

Replication is not consensus-based. There is no Raft; `-replicationFactor=N` on `vminsert` simply writes each series to N storage nodes, and correctness on read depends on de-duplication (`-dedup.minScrapeInterval`). Because `vmstorage` nodes are independent, **adding a node does not rebalance existing data** — new series land on the enlarged set, old data stays where it was until it ages out of retention.

Around the database sits a family of single-purpose binaries in the same repo: **`vmagent`** (scrape + buffer + remote-write with on-disk queue), **`vmalert`** (Prometheus-compatible recording and alerting rules), **`vmauth`** (auth/routing proxy), **`vmbackup`/`vmrestore`** (snapshot-based backups to S3/GCS), and **`vmctl`** (migrate data from Prometheus/InfluxDB/other VM). Cluster multi-tenancy is encoded in the URL path (`/insert/<accountID>:<projectID>/...`); in OSS there is no cross-tenant query — a global view across tenants is an Enterprise feature.

## Production Notes

- **Cluster scaling is manual and asymmetric.** Because `vmstorage` never rebalances, capacity planning matters up front. The common recovery from a hot/full node is to add capacity and wait out retention, or to re-ingest — not a click-to-rebalance.
- **Replication ≠ backups, and ≠ HA quorum.** `replicationFactor` protects against node loss but has no quorum; a network partition can accept writes on both sides. Real durability still needs `vmbackup` snapshots to object storage.
- **De-duplication is load-bearing.** Running two Prometheus/`vmagent` replicas into VM for HA requires `-dedup.minScrapeInterval` set to the scrape interval, or you get double-counted rates.
- **No downsampling in OSS.** Long retention of high-resolution data is a disk-cost problem the open-source build won't solve for you; the usual mitigations are shorter retention, coarser scrape intervals, or `vmalert` recording rules that pre-aggregate.
- **MetricsQL vs PromQL drift.** MetricsQL changes defaults (e.g. how `rate`/`increase` extrapolate at series edges, handling of staleness). Migrating dashboards from Prometheus can shift graph values slightly; validate rather than assume parity[^4].
- **High churn is the real cost driver.** VM handles high cardinality well, but constantly-replaced series (churn) inflate the inverted index and RAM. The `-search.maxUniqueTimeseries` and cardinality-limiter flags cap blast radius; `/api/v1/status/tsdb` surfaces top offenders.
- **The project moves fast.** The changelog cadence is high and upgrades are usually smooth, but "read the CHANGELOG before bumping" is real advice here — flag defaults and semantics do change between minor releases.

## When to Use / When Not

**Use when:**
- You run Prometheus and need cheaper, longer-retention long-term storage without a Cassandra/object-store stack.
- RAM and disk cost of your metrics is the pain point; VM's compression and memory profile are its core advantage.
- You want a single Go binary (or a small set of them) with no external database dependency.
- You need to consolidate InfluxDB/Graphite/OpenTSDB/OTel ingestion behind one PromQL-queryable store.

**Avoid when:**
- You need downsampling, per-dataset retention, or anomaly detection without paying for Enterprise.
- You require automatic cluster rebalancing or consensus-based replication (VM has neither).
- You depend on exact PromQL semantics and can't tolerate MetricsQL's default differences.
- Your data is general-purpose (relational joins, events, non-metric) rather than numeric time series — a TSDB is the wrong shape.

## Alternatives

- prometheus/prometheus — use when you want the CNCF-standard single-node metrics store and don't yet need long-term-storage scale or a second cluster tier.
- grafana/mimir — use when you want horizontally scalable, object-storage-backed, multi-tenant Prometheus with automatic sharding (the actively developed Cortex successor).
- thanos-io/thanos — use when you're already Prometheus-centric and want a global query view plus S3/GCS long-term storage bolted onto existing servers.
- influxdata/influxdb — use when you need a general-purpose TSDB with its own query language and push-based ingestion beyond the metrics/PromQL model.
- timescale/timescaledb — use when you want SQL, Postgres tooling, and relational joins alongside time series rather than a metrics-only engine.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-09 | Repository created; project started by Aliaksandr Valialkin[^1]. |
| v1.x | 2019 | First public single-node releases under Apache-2.0. |
| cluster | ~2020 | Cluster version (`vminsert`/`vmstorage`/`vmselect`) published as open source. |
| vmagent | ~2020 | Remote-write scraping agent with on-disk buffering added. |
| stream agg | ~2023 | Native stream aggregation (StatsD-style pre-aggregation) added. |
| ongoing | 2026 | Actively maintained; high release cadence, daily commits[^2]. |

## References

[^1]: VictoriaMetrics repository metadata (created 2018-09-30). https://github.com/VictoriaMetrics/VictoriaMetrics
[^2]: GitHub API — 17,330 stars, 1,686 forks, last push 2026-07-14 (fetched for this page).
[^3]: VictoriaMetrics Enterprise features (downsampling, multiple retentions, backup automation, anomaly detection). https://docs.victoriametrics.com/victoriametrics/enterprise/
[^4]: MetricsQL — differences from PromQL. https://docs.victoriametrics.com/victoriametrics/metricsql/
[^5]: Cluster VictoriaMetrics architecture (vminsert/vmstorage/vmselect, replication, multi-tenancy). https://docs.victoriametrics.com/victoriametrics/cluster-victoriametrics/

## Tags

go, time-series-database, tsdb, monitoring, observability, prometheus, promql, metrics, grafana, opentelemetry, database
