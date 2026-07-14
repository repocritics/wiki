# apache/druid

> A distributed, column-oriented real-time analytics database built for sub-second aggregation queries over large event streams at high concurrency.

[GitHub repo](https://github.com/apache/druid) ·
[Official website](https://druid.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/druid/blob/master/LICENSE)

## Overview

Druid is an OLAP data store designed for a specific shape of workload: fast filtered aggregations (GROUP BY over time ranges, top-N, timeseries) served to interactive dashboards and operational tooling, with high query concurrency and streaming ingestion. It was built at Metamarkets in 2011 to power an ad-analytics UI, open-sourced in 2012, and became a top-level Apache project in 2021[^1]. It is not a general-purpose SQL database and not a transactional one — it is a purpose-built aggregation engine.

The defining tradeoff is **specialization for operational overhead**. Druid gets its speed from time-partitioned immutable columnar segments, per-column bitmap indexes, and optional ingestion-time rollup (pre-aggregation). In exchange, a Druid cluster is genuinely a distributed system with many always-on service types plus three external dependencies (deep storage, a metadata store, and ZooKeeper). There is no single-binary production mode; the smallest realistic cluster still involves coordinating several roles. Teams adopt Druid when the query characteristics justify that operational cost, and regret it when they wanted "a fast database" without the moving parts.

Druid's closest peers are Apache Pinot and ClickHouse. Pinot shares almost the same architecture and use case; ClickHouse reaches similar query speeds with far simpler operations but weaker streaming-ingestion and high-concurrency stories. Choosing among them is usually the real decision, not Druid-vs-a-warehouse.

## Getting Started

The quickstart runs a single-machine micro-cluster (all services in one JVM group) for evaluation only:

```bash
# JDK 17+ required at runtime; JDK 21 or 25 to build from source
curl -O https://dlcdn.apache.org/druid/34.0.0/apache-druid-34.0.0-bin.tar.gz
tar -xzf apache-druid-34.0.0-bin.tar.gz
cd apache-druid-34.0.0
./bin/start-druid            # starts coordinator, overlord, broker, historical, router
# web console at http://localhost:8888
```

Ingest a batch file with SQL-based (MSQ) ingestion via the console or the API:

```sql
-- Multi-Stage Query engine: SQL INSERT reads, shuffles, and writes segments
INSERT INTO wikipedia
SELECT
  TIME_PARSE("timestamp") AS __time,   -- every table is partitioned on __time
  page, channel, countryName,
  SUM("added") AS chars_added
FROM TABLE(EXTERN(
  '{"type":"http","uris":["https://druid.apache.org/data/wikipedia.json.gz"]}',
  '{"type":"json"}'
))
GROUP BY 1, 2, 3, 4
PARTITIONED BY DAY
```

## Architecture / How It Works

Data in Druid lives as **segments**: immutable, compressed, columnar files scoped to a time interval and a version. Each column is stored separately; string dimensions carry a bitmap index (Roaring/Concise) so filters resolve to a set of matching rows without a scan. Metrics are stored for aggregation. If **rollup** is enabled, rows sharing the same dimension values within a time granularity are pre-summed at ingestion, trading raw-row fidelity for smaller segments and faster queries.

A cluster is a set of cooperating service roles, conventionally grouped:

- **Master** — the *Coordinator* manages segment placement, balancing, and retention/drop rules; the *Overlord* assigns and tracks ingestion tasks.
- **Query** — the *Broker* fan-outs a query to the data servers holding relevant segments, merges partial results, and returns them; the optional *Router* is an API/console gateway.
- **Data** — *Historicals* memory-map and serve immutable segments pulled from deep storage; *MiddleManagers* (or the newer *Indexer*) run ingestion tasks and serve not-yet-published real-time segments.

Three external dependencies are mandatory: **deep storage** (S3/GCS/HDFS/local — the permanent segment store and source of truth), a **metadata store** (PostgreSQL/MySQL; Derby only for the quickstart) holding segment records, task state, and config, and **ZooKeeper** for service discovery and coordination[^2]. Historicals hold no durable state — they are a cache of deep storage, which is why a lost Historical is a re-download, not data loss.

Queries come in two dialects. **Druid SQL** is an Apache Calcite planner that compiles to Druid's **native JSON queries** (timeseries, topN, groupBy, scan). Since Druid 24.0 the **Multi-Stage Query (MSQ)** engine adds a shuffle-capable execution layer used for SQL-based batch ingestion (`INSERT`/`REPLACE`) and heavier reporting queries[^3]. Streaming ingestion runs through **supervisors** that pull from Apache Kafka or Amazon Kinesis with exactly-once semantics, building real-time segments that are periodically handed off to Historicals.

## Production Notes

- **Segment sizing is the master tuning knob.** Aim for roughly 5 million rows or a few hundred MB per segment. Too many tiny segments overload the Coordinator and Broker with metadata and per-segment query overhead; oversized segments hurt parallelism. **Auto-compaction** exists to merge small segments after the fact and should be configured early, not discovered late.
- **Memory model is off-heap-heavy.** Historicals rely on the OS page cache for memory-mapped segments, so provisioning is "segment working set should fit in free RAM," not just heap. Processing uses off-heap merge buffers (`druid.processing.buffer.sizeBytes`, `numMergeBuffers`); undersizing these causes `Resource limit exceeded` on groupBy/MSQ queries.
- **Updates and deletes are not row-level.** Correcting data means reindexing or `REPLACE`-ing whole time intervals, or running compaction. There is no cheap `UPDATE WHERE id = ...`. Design ingestion to be idempotent over time ranges.
- **JOINs are limited and expensive.** Historically only broadcast joins (small right side) are supported; large fact-to-fact joins are impractical. The idiomatic pattern is denormalize at ingestion, or use **lookups** for dimension enrichment. Teams arriving from a warehouse mindset hit this wall first.
- **High-cardinality dimensions are costly** in both segment size and query memory; sketch-based approximations (`APPROX_COUNT_DISTINCT` via HLL/Theta sketches, `DS_QUANTILE`) are the intended escape hatch and are widely used in practice.
- **Many always-on nodes, no serverless mode.** Deep storage, metadata DB, ZooKeeper, and each service role are separate operational surfaces. Rolling upgrades have a recommended service order (Historicals and Overlord/MiddleManagers before Coordinator/Broker); read the upgrade notes per release.
- **The web console is a real operational tool** (ingestion wizard, segment/task views, query workbench backed by SQL system tables), which offsets some of the complexity but does not remove it.

## When to Use / When Not

**Use when:**
- You need sub-second GROUP BY / topN over billions of rows powering an interactive UI, at high concurrency.
- You have continuous event streams (clickstream, metrics, telemetry, ad events) and want streaming + batch ingestion in one store.
- Your queries are time-scoped aggregations rather than point lookups, ad-hoc JOINs, or full-row retrieval.

**Avoid when:**
- You want a fast analytical database with minimal ops — ClickHouse (single node) or a managed warehouse is far less to run.
- Your workload needs frequent row-level updates/deletes, multi-table JOINs, or transactions.
- Your data volume is small enough that Postgres or DuckDB answers the queries — Druid's fixed operational cost dominates at low scale.

## Alternatives

- apache/pinot — nearly the same architecture and use case; consider when you need lower query latency, upserts, or tighter real-time freshness.
- clickhouse/clickhouse — columnar OLAP with dramatically simpler operations; use when a single node or small cluster suffices and you don't need Druid's streaming/high-concurrency machinery.
- apache/doris / StarRocks/starrocks — MPP analytical databases with a MySQL protocol and stronger JOIN support; use when you want warehouse-style SQL over denormalization.
- elastic/elasticsearch — overlapping for log/event analytics with full-text search; use when search and flexible schemas matter more than aggregation throughput.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2011 | Built at Metamarkets to back an ad-analytics UI[^1]. |
| — | 2012-10 | Open-sourced (GPL; later relicensed to Apache 2.0). |
| — | 2018-02 | Entered the Apache Incubator[^1]. |
| — | 2021 | Graduated to a top-level Apache project[^1]. |
| 0.23.0 | 2022-06 | Final release of the 0.x line. |
| 24.0.0 | 2022-09 | Switched to `MAJOR.MINOR.PATCH` versioning; introduced the Multi-Stage Query (MSQ) engine and SQL-based ingestion[^3]. |
| 34.0.0 | 2026 | Recent release line; JDK 17+ runtime, JDK 21/25 to build[^4]. |

## References

[^1]: Apache Druid — project background and history. https://druid.apache.org/docs/latest/design/
[^2]: Apache Druid docs, "Architecture" (processes, deep storage, metadata store, ZooKeeper dependencies). https://druid.apache.org/docs/latest/design/architecture.html
[^3]: Apache Druid docs, "Multi-stage query architecture" (MSQ, SQL-based ingestion). https://druid.apache.org/docs/latest/multi-stage-query/
[^4]: Apache Druid README and build guide (JDK requirements). https://github.com/apache/druid/blob/master/README.md

## Tags

java, olap, analytics-database, columnar, real-time, distributed-systems, streaming, time-series, apache, sql, big-data
