# risingwavelabs/risingwave

> A Postgres-wire-compatible streaming database in Rust — incrementally maintained materialized views with state kept in object storage instead of local disk.

[GitHub repo](https://github.com/risingwavelabs/risingwave) ·
[Official website](https://risingwave.com) ·
[License: Apache-2.0](https://github.com/risingwavelabs/risingwave/blob/main/LICENSE)

## Overview

RisingWave is a distributed SQL streaming database written in Rust. Its core abstraction is the incrementally maintained materialized view: you define a view with SQL over one or more streaming sources, and RisingWave keeps the result up to date as new events arrive, recomputing only the affected rows rather than re-running the query. It speaks the PostgreSQL wire protocol, so `psql`, JDBC, and most Postgres client libraries connect without a custom driver[^1]. The project was open-sourced in 2022 by Singularity Data (since renamed RisingWave Labs), founded by Yingjun Wu, formerly of AWS Redshift and IBM Research[^2].

The positioning has shifted over the project's life. Early messaging framed it as a "cloud-native streaming database" and a SQL-first alternative to Apache Flink; the current README markets it as an "event streaming platform for agentic AI"[^1]. The marketing layer changes faster than the engine underneath, which has consistently been the same thing: a stream processor that replaces the Debezium + Kafka + Flink + serving-database stack with one system exposing tables and materialized views over SQL.

The defining architectural bet is **state on object storage**. Streaming operators (joins, aggregations, windows) hold large amounts of intermediate state; Flink keeps this on local disk via RocksDB, RisingWave keeps it in S3-compatible storage through its own LSM-tree engine. This trades single-digit-millisecond local state access for elastic scaling, fast failure recovery, and roughly 100x cheaper storage — with a local disk cache to claw back latency for hot data[^1]. That tradeoff is the thing to evaluate before adopting.

## Getting Started

```shell
# single-node playground (installs a local binary)
curl -L https://risingwave.com/sh | sh
risingwave
```

Connect with any Postgres client and build a streaming pipeline:

```sql
-- 1. define a source (Kafka topic of JSON events)
CREATE SOURCE clicks (
  user_id INT,
  url     VARCHAR,
  ts      TIMESTAMPTZ
) WITH (
  connector = 'kafka',
  topic     = 'clicks',
  properties.bootstrap.server = 'kafka:9092'
) FORMAT PLAIN ENCODE JSON;

-- 2. materialized view — incrementally maintained, always fresh
CREATE MATERIALIZED VIEW clicks_per_user AS
SELECT user_id, count(*) AS n
FROM clicks
GROUP BY user_id;

-- 3. query the view like a normal table (served from internal row store)
SELECT * FROM clicks_per_user WHERE user_id = 42;
```

For Docker Compose, Kubernetes (Helm or the operator), and the managed RisingWave Cloud, see the deployment docs[^3].

## Architecture / How It Works

RisingWave separates into four node roles, coordinated through a control plane[^4]:

1. **Meta node** — the control plane. Schedules streaming jobs, tracks cluster membership, drives checkpointing (barrier injection), and manages catalog metadata.
2. **Frontend node** — parses SQL, plans queries, and serves the Postgres wire protocol. Stateless; scale for connection concurrency.
3. **Compute node** — runs the actual streaming operators and batch (ad-hoc) queries. This is where materialized-view state lives in memory + local cache.
4. **Compactor node** — background LSM compaction of state written to object storage. Under-provisioning compactors is a classic failure mode (see below).

State is written through **Hummock**, RisingWave's LSM-tree state store backed by S3-compatible object storage. Consistency is exactly-once, achieved with **checkpoint barriers** flowing through the dataflow graph (a Chandy–Lamport-style snapshot, the same family of algorithm Flink uses)[^4]. A barrier interval defines how often state is checkpointed to object storage; it is also the coarse bound on end-to-end freshness under load.

The **row store** serves point and range queries on materialized views at low latency; the **Iceberg layer** is the durable, open-format side. Recent versions can host an Apache Iceberg REST catalog directly, write MV output to Iceberg tables with automated compaction and snapshot cleanup, and read them back via an embedded Apache DataFusion engine, so the same data is also readable by Spark, Trino, or DuckDB[^1]. The two stores are complementary, not redundant: row store for serving, Iceberg for retention and analytics.

Postgres compatibility is at the **wire-protocol and dialect level, not full semantic parity**. You get Postgres syntax, types, and many functions, plus streaming extensions (`CREATE SOURCE`, `CREATE SINK`, `EMIT ON WINDOW CLOSE`, watermarks). You do not get a general-purpose OLTP database — there is no MVCC transactional workload story, and behaviors around updates/deletes, isolation, and some functions diverge from real Postgres.

## Production Notes

**Compactor sizing is the first operational trap.** Because all state goes through an LSM tree on object storage, write-heavy pipelines generate continuous compaction work. If compactor nodes are under-provisioned, unmerged SST files pile up, read amplification grows, and query and streaming latency degrade until the cluster falls behind. Compactor CPU/count is a primary capacity-planning dimension, not an afterthought.

**Object-storage state means latency depends on cache hit rate.** Cold state reads hit S3 (tens of milliseconds); hot state is served from memory and the local disk cache. Large stateful operators (wide joins, high-cardinality aggregations, long windows) that exceed cache size will show tail-latency and throughput sensitivity. Size the elastic disk cache (local SSD/EBS) for your working set; do not assume the advertised 10–20 ms p99 without it.

**Materialized-view backfill is expensive.** Creating an MV over historical data (or over an upstream table with existing rows) triggers a backfill that reads and processes the full history before the view goes live. On large sources this can take a long time and consume significant state; plan MV creation as a real operation, not an instant DDL.

**Freshness is bounded by the barrier interval and backpressure.** The sub-100 ms freshness figure holds for healthy pipelines. Under backpressure (slow sink, undersized compute, S3 throttling), barriers stall and end-to-end lag grows. Monitor barrier latency and object-storage error rates.

**Upgrades can require recreating streaming jobs.** Internal streaming-state and plan formats evolve across releases, and RisingWave does not guarantee transparent in-place migration of running materialized views across every version boundary. Read the release notes for state-compatibility caveats; some upgrades in practice mean recreating (and re-backfilling) MVs. Test upgrades on a staging cluster.

**Connectors and CDC.** Kafka, Pulsar, Kinesis, and Postgres/MySQL CDC are the well-trodden paths. CDC correctness depends on upstream configuration (replication slots, `wal_level`, retention); a dropped slot or truncated WAL breaks the stream and forces a re-snapshot. Sink delivery guarantees vary per connector — confirm whether a given sink is exactly-once or at-least-once for your target.

**Telemetry is on by default.** RisingWave collects anonymized usage statistics (via Scarf and its own reporting) unless you opt out; relevant in regulated or air-gapped environments[^1].

## When to Use / When Not

**Use when:**
- You want streaming pipelines and always-fresh materialized views expressed in SQL, without operating a JVM stream processor.
- You want to consolidate CDC + transport + processing + a serving store into one system and query results over the Postgres protocol.
- Elastic scaling and fast (seconds) failure recovery matter more than absolute lowest state-access latency.
- You want streaming output to land in Apache Iceberg in an open format for downstream engines.

**Avoid when:**
- You need a general-purpose transactional (OLTP) Postgres — this is not that, despite the wire compatibility.
- Your logic doesn't fit relational streaming SQL and needs Flink's low-level DataStream API or custom operators.
- Your state working set is small and latency-critical, and object-storage-backed state buys you nothing over a local-disk processor.
- You need a large, battle-tested connector ecosystem and years of production hardening across every edge — Flink's ecosystem is broader.

## Alternatives

- apache/flink — the mature incumbent; more flexible (DataStream + SQL), broader connectors, but JVM operational burden and local-disk state to manage.
- MaterializeInc/materialize — the closest peer: incremental view maintenance over SQL, built on Timely/Differential Dataflow rather than an LSM-on-S3 design.
- feldera/feldera — incremental computation engine (DBSP theory); strong on correctness of incremental results, younger ecosystem.
- apache/spark — Structured Streaming is micro-batch, not true per-event incremental; use it when you already run Spark and can tolerate batch latency.
- timeplus-io/proton — streaming SQL built on the ClickHouse engine; pick it when analytical scan throughput matters more than maintained views.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2022-01 | Repository created; open-sourced by Singularity Data[^2]. |
| 1.0 | 2023 | First GA release; project rebranded to RisingWave Labs. |
| — | 2024 | Deepened Apache Iceberg integration (sink, then native catalog/reads). |
| — | 2026 | Repositioned as "event streaming for agentic AI"; MCP server, CLI, and Skills for agent access[^1]. |

Exact minor-version dates are omitted where not verified; consult the GitHub releases page for the authoritative changelog.

## References

[^1]: RisingWave README and product docs — https://github.com/risingwavelabs/risingwave and https://docs.risingwave.com/
[^2]: RisingWave Labs (formerly Singularity Data), company background — https://risingwave.com/about/
[^3]: RisingWave deployment guide (Docker Compose, Kubernetes/Helm, Operator, Cloud) — https://docs.risingwave.com/get-started/quickstart
[^4]: RisingWave developer guide / architecture — https://risingwavelabs.github.io/risingwave/

## Tags

rust, streaming-database, stream-processing, materialized-views, postgresql, sql, apache-iceberg, cdc, kafka, real-time-analytics, object-storage, data-engineering
