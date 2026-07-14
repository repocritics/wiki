# influxdata/influxdb

> Time-series database for metrics, events, and real-time analytics — now on its third engine, a Rust rewrite atop Apache Arrow, DataFusion, and Parquet.

[GitHub repo](https://github.com/influxdata/influxdb) ·
[Official website](https://influxdata.com) ·
[License: MIT OR Apache-2.0](https://github.com/influxdata/influxdb/blob/main/LICENSE)

## Overview

InfluxDB is a purpose-built time-series datastore for metrics, events, and monitoring workloads: high-volume timestamped writes, retention windows, and fast recent-data queries for dashboards and alerting[^1]. It has been one of the most-recognized names in the time-series space since the mid-2010s, and this single repository is unusual in hosting three architecturally distinct products across three long-lived branches.

The defining fact about InfluxDB is that it has been rewritten twice. The `main` branch (this default) is **InfluxDB 3 Core**, a Rust codebase built on the Apache Arrow / DataFusion / Parquet stack (the "FDAP" stack: Flight, DataFusion, Arrow, Parquet)[^2]. The `main-2.x` branch is **InfluxDB 2.x**, written in Go, centered on the Flux scripting language and a bundled UI. The `master-1.x` branch is **InfluxDB 1.x**, also Go, built on the TSM (Time-Structured Merge Tree) storage engine and queried with InfluxQL. The GitHub-reported primary language flipped from Go to Rust when v3 became the default branch; older tooling and much of the wider InfluxData ecosystem (Telegraf, the v1/v2 servers) remain Go.

This lineage is the central tension. Each rewrite chased a real limitation of the prior engine — chiefly **series cardinality**, the number of unique tag combinations, which the TSM engine of v1/v2 held largely in memory and which caused the ecosystem's most infamous out-of-memory failures. v3's columnar, object-store-backed design is specifically meant to make cardinality a non-issue. The cost is migration pain: the query languages, storage format, and operational model changed each time, and Flux — the flagship of v2 — is not carried forward into v3.

## Getting Started

Docker, Debian/RPM packages, and tarballs are published on InfluxData's downloads portal[^3]. Quickest path (v3 Core):

```bash
docker run -it -p 8181:8181 \
  quay.io/influxdb/influxdb3-core:latest \
  serve --node-id host01 --object-store memory
```

Write with line protocol and query with SQL over the HTTP API on port 8181:

```bash
# Write a point (line protocol: measurement,tags fields timestamp)
curl -X POST "http://localhost:8181/api/v3/write_lp?db=sensors" \
  --data-binary "cpu,host=host01 usage=0.64 $(date +%s)000000000"

# Query with SQL
curl "http://localhost:8181/api/v3/query_sql" \
  --data '{"db":"sensors","query":"SELECT * FROM cpu ORDER BY time DESC LIMIT 10"}'
```

The write path is compatible with the InfluxDB 1.x and 2.x write APIs, and v3 still speaks InfluxQL alongside SQL, easing partial migrations[^1].

## Architecture / How It Works

InfluxDB 3 is a **diskless, object-store-native** design[^1]:

- **Ingest** — writes arrive as line protocol, are validated, and buffered in memory as Arrow record batches, with a write-ahead log for durability.
- **Persist** — buffered data is flushed to **Parquet** files on object storage (S3, Azure Blob, GCS) or local disk. Storage and compute are decoupled; the database itself is largely stateless beyond its catalog and WAL.
- **Query** — **Apache DataFusion** is the SQL engine; queries fan out over Parquet files plus the in-memory buffer. InfluxQL is translated onto the same engine. Results are served over the HTTP query API and Arrow **Flight SQL**.
- **Compact** — background compaction merges small Parquet files into larger, sorted ones to keep query planning efficient (this is the tier that materially differs between Core and Enterprise).

An **embedded Python VM** runs plugins and triggers in-process, letting you transform or route data on write without an external stream processor[^1].

The earlier engines are a different world: v1/v2 store data in the TSM engine — a columnar-ish LSM variant with an in-memory inverted index (TSI) mapping series keys. That index is what made high cardinality expensive. v2 layered on Flux (a functional data-scripting language), a task scheduler, and a web UI as a single monolith. None of that internal machinery is shared with v3; they are separate programs behind one brand.

## Production Notes

**Know which version you are actually running.** "InfluxDB" in a blog post, Stack Overflow answer, or old Terraform module may mean any of three incompatible systems. Query language, storage layout, ports, and client libraries differ across v1/v2/v3. Pin the major version in every runbook.

**Cardinality is the historical footgun.** On v1/v2, putting high-cardinality values (user IDs, request IDs, UUIDs) into *tags* rather than *fields* inflates the series index and drives OOM crashes. v3 is designed to tolerate this via Parquet columns, but the tag-vs-field modeling discipline still matters for query performance. Audit tag design before it becomes a memory incident.

**Core is deliberately limited; Enterprise is the paid tier.** InfluxDB 3 Core (this repo, open source) targets recent-data, single-node use — real-time monitoring and last-value dashboards. Historical querying at scale, high availability, read replicas, and richer compaction are reserved for the commercial **InfluxDB 3 Enterprise** (and the hosted Cloud Dedicated / Serverless products, which run the same v3 engine)[^4]. Do not size a long-retention analytics workload on Core without confirming its current query-window and retention constraints against the release notes — these limits have shifted between releases.

**Flux is a migration dead end.** Teams that invested heavily in Flux dashboards and tasks on v2 cannot lift them onto v3, which speaks SQL and InfluxQL instead. Flux is in maintenance on the `main-2.x` line. Plan a rewrite, not an upgrade, when moving v2 → v3.

**Object storage is now a dependency for real deployments.** The diskless model shines with S3-class storage, but that adds latency characteristics and request-cost considerations absent from the old single-binary-on-a-local-disk v1 experience. Local disk works for small/edge cases.

## When to Use / When Not

**Use when:**
- You ingest high-volume timestamped data and mostly query recent windows (server/app/network monitoring, IoT sensors, trading telemetry).
- You want line-protocol writes and a managed-or-self-hosted spectrum from a single-binary edge node up to a hosted cluster.
- You value Parquet-on-object-store so your data stays in an open, queryable-by-other-tools format.

**Avoid when:**
- You need mature, stable v3 features today for deep historical analytics on a single open-source node — Core's limits may push you to the paid tier.
- Your team is committed to Flux — v3 abandons it.
- Your workload is relational/OLAP with joins across many dimensions rather than time-first — a general columnar engine fits better.
- You want a metrics stack with built-in alerting and a pull model — Prometheus is a closer match.

## Alternatives

- timescale/timescaledb — a PostgreSQL extension; use when you want full SQL, joins, and relational features alongside time-series and prefer the Postgres ecosystem.
- prometheus/prometheus — use for pull-based metrics monitoring and alerting where the data model is numeric metrics, not arbitrary events.
- questdb/questdb — use when raw single-node ingest throughput and SQL with time extensions are the priority.
- VictoriaMetrics/VictoriaMetrics — use as a Prometheus-compatible, cost-efficient long-term metrics store that handles high cardinality well.
- ClickHouse/ClickHouse — use when the workload is large-scale analytical (OLAP) beyond time-series and you want a general columnar warehouse.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2013 | Initial open-source release, written in Go[^5]. |
| 1.0 | 2016-09 | TSM storage engine, InfluxQL, single-binary server[^5]. |
| 1.8 | 2020 | Long-lived 1.x line; adds Flux preview, remains widely deployed. |
| 2.0 | 2020-11 | Go rewrite of the platform: Flux language, bundled UI, tasks, buckets[^6]. |
| 3.0 engine | 2023 | "IOx" Rust engine (Arrow/DataFusion/Parquet) ships first in hosted Cloud products[^2]. |
| 3 Core GA | 2025-04 | Open-source v3 Core + Enterprise generally available; `main` becomes v3[^4]. |

## References

[^1]: InfluxDB 3 Core README and feature list. https://github.com/influxdata/influxdb
[^2]: InfluxData, "InfluxDB IOx / the FDAP stack" (Apache Arrow, DataFusion, Flight, Parquet). https://www.influxdata.com/blog/flight-datafusion-arrow-parquet-fdap-architecture-influxdb/
[^3]: InfluxData downloads portal. https://portal.influxdata.com/downloads/
[^4]: InfluxData, "InfluxDB 3 Core and Enterprise are now generally available" — 2025-04. https://www.influxdata.com/blog/influxdb-3-oss-ga/
[^5]: InfluxDB v1.x branch and release tags. https://github.com/influxdata/influxdb/tree/master-1.x
[^6]: InfluxDB v2.x branch. https://github.com/influxdata/influxdb/tree/main-2.x

## Tags

time-series, database, rust, go, monitoring, metrics, observability, apache-arrow, datafusion, parquet, iot, sql
