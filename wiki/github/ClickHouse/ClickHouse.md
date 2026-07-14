# ClickHouse/ClickHouse

> Column-oriented SQL database built for real-time analytics over very large tables — fast at aggregation scans, deliberately weak at point mutations.

[GitHub repo](https://github.com/ClickHouse/ClickHouse) ·
[Official website](https://clickhouse.com) ·
[License: Apache-2.0](https://github.com/ClickHouse/ClickHouse/blob/master/LICENSE)

## Overview

ClickHouse is a column-oriented OLAP database management system written in C++. It began inside Yandex as the storage engine behind Yandex.Metrica (a web-analytics product) and was open-sourced in June 2016[^1]. In 2021 the project spun out into an independent company, ClickHouse Inc., which now maintains the open-source codebase and sells a managed offering (ClickHouse Cloud). The original author, Alexey Milovidov, remains the technical lead.

The system is optimized for one workload: aggregating over billions of rows with a handful of columns touched per query. It stores each column separately, compresses it heavily, and runs a vectorized execution engine that processes data in blocks rather than row-at-a-time, using SIMD where available. On the workloads it targets — event logs, metrics, traces, clickstream, observability — it is among the fastest systems available, and it scales from a single binary on a laptop to sharded, replicated clusters.

The defining tradeoff is that ClickHouse is not a transactional database and does not pretend to be one. There are no cross-table transactions, no enforced foreign keys, no per-row locking, and updates/deletes are asynchronous rewrites rather than in-place edits. It is an append-optimized analytical store; treating it like PostgreSQL is the single most common source of production pain. The `®` in the project's own naming reflects that "ClickHouse" is a registered trademark of the company, not just a project name.

## Getting Started

```bash
curl https://clickhouse.com/ | sh
./clickhouse server        # start the server
./clickhouse client        # connect an interactive SQL client
```

```sql
-- MergeTree is the default engine. ORDER BY defines the sparse primary
-- index (also the sort order on disk) — choose it to match query filters.
CREATE TABLE events
(
    event_time DateTime,
    user_id    UInt64,
    country    LowCardinality(String),
    url        String
)
ENGINE = MergeTree
ORDER BY (country, event_time);

INSERT INTO events VALUES
    ('2026-07-14 10:00:00', 42, 'KR', '/home');

SELECT country, count() AS c
FROM events
WHERE event_time >= now() - INTERVAL 1 DAY
GROUP BY country
ORDER BY c DESC;
```

## Architecture / How It Works

**MergeTree** is the storage engine family that everything else builds on. Inserts are written as immutable *parts* (sorted, compressed column files); a background process continually merges small parts into larger ones, LSM-tree style. There is no per-row B-tree index — instead a *sparse* primary index stores one mark per N rows (8192 by default), so lookups scan a granule, not a single row. This is why `ORDER BY` on table creation matters more than any other decision: it is both the on-disk sort order and the index.

**Vectorized execution.** Queries run over blocks of columns (default ~65k rows), which keeps hot loops cache-friendly and SIMD-vectorizable. Aggregate functions, filters, and joins are all block-oriented.

**Compression** is per-column, defaulting to LZ4 (fast) with ZSTD available for higher ratios. Because a column holds homogeneous data, compression ratios are typically far better than row stores. Specialized codecs (`Delta`, `DoubleDelta`, `Gorilla`, `T64`) target time-series and monotonic data.

**Replication and sharding are separate concerns.** `ReplicatedMergeTree` handles replicas via a coordination service — historically ZooKeeper, now **ClickHouse Keeper**, a C++ Raft reimplementation that removes the JVM dependency. The `Distributed` engine handles sharding: it is a query router/fan-out layer over local tables on each shard, not a storage engine itself.

**Materialized views** are insert-time triggers, not lazily-refreshed snapshots. A row inserted into the source table is transformed and written to the view's target table at insert time — commonly used with `AggregatingMergeTree` / `SummingMergeTree` to maintain rollups. Getting this mental model wrong (expecting Postgres-style refresh) is a frequent surprise.

**Integration engines and table functions** let ClickHouse read/write external systems directly: `Kafka`, `MySQL`, `PostgreSQL`, `S3`, `url`, `file`, and increasingly data-lake formats (Iceberg, Delta, Hudi) reflecting the project's push toward lakehouse use cases.

## Production Notes

**The "too many parts" problem.** Frequent small inserts create many small parts and overwhelm background merging, eventually throwing `TOO_MANY_PARTS`. The fix is to batch: insert in large blocks (tens of thousands of rows), or enable **async inserts** so the server buffers and flushes. Do not do one-row-per-request inserts from application code.

**Mutations are expensive.** `ALTER TABLE ... UPDATE/DELETE` are asynchronous *mutations* that rewrite entire affected parts in the background — they are not OLTP updates and can take minutes to hours on large tables. Lightweight `DELETE` (and, in newer versions, lightweight updates) mitigate this for some cases, but the design assumption remains append-mostly. Model data so you rarely need to mutate; use `ReplacingMergeTree` or versioned rows for "latest value" semantics.

**JOIN semantics differ from row stores.** The classic hash join loads the right-hand table into memory, so joining two large tables can OOM. Order tables so the smaller side is on the right, use dictionaries for dimension lookups, or denormalize. Newer join algorithms (`full_sorting_merge`, `grace_hash`, partial-merge) reduce but do not eliminate the need to think about join memory.

**Memory and high-cardinality GROUP BY.** Aggregations build hash tables in RAM; high-cardinality `GROUP BY` can exhaust memory and hit `MEMORY_LIMIT_EXCEEDED`. Controls: `max_memory_usage`, spill-to-disk via `max_bytes_before_external_group_by`, and `LowCardinality` column types for repetitive strings.

**Consistency is eventual across replicas.** `ReplicatedMergeTree` replication is asynchronous by default; a read from a lagging replica may not see a just-inserted row unless you use `insert_quorum` / `select_sequential_consistency`. There is no multi-statement transaction across tables.

**Versioning and upgrades.** Releases use calendar versioning `YY.M` (e.g., 25.x, 26.x), with periodic LTS releases for teams that want a slower cadence. The release pace is fast; pin a version, read changelogs for settings-default changes, and test upgrades — default behaviors do shift between releases.

## When to Use / When Not

**Use when:**
- You run analytical scans/aggregations over large, append-heavy datasets (logs, metrics, events, observability, product analytics).
- You want single-system scaling from one node to a replicated cluster with SQL as the interface.
- You need high ingest throughput and strong compression more than you need row-level mutability.

**Avoid when:**
- You need OLTP: frequent single-row updates/deletes, cross-table transactions, or strong read-after-write consistency.
- Your queries are point lookups by primary key returning one row — a row store (Postgres, MySQL) fits better.
- You need enforced constraints/foreign keys or a normalized relational model with heavy joins between large tables.
- Your data is small enough to fit an embedded analytical engine — DuckDB avoids the operational overhead.

## Alternatives

- duckdb/duckdb — use instead when the data is single-node / embedded and you don't need a server or a cluster.
- apache/druid — use instead when you need a purpose-built real-time ingestion + query cluster and accept heavier operational complexity.
- apache/pinot — use instead when the priority is ultra-low-latency user-facing analytics with tight p99 SLAs.
- StarRocks/starrocks — use instead when you want ClickHouse-like speed with a stronger built-in join/optimizer story for star-schema BI.
- timescale/timescaledb — use instead when you want time-series analytics but need to stay inside PostgreSQL and its transactional guarantees.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-06 | Open-sourced by Yandex (Apache-2.0)[^1]. |
| 19.x | 2019 | Switched to `YY.M` calendar versioning; rapid feature growth. |
| 21.x | 2021 | ClickHouse Inc. spun out as an independent company[^2]. |
| 21.x | 2021 | ClickHouse Keeper introduced as a C++ ZooKeeper replacement. |
| 22.x | 2022 | Lightweight `DELETE`, broader S3 / data-lake integration. |
| 26.5 | 2026-05 | Recent monthly release (community release call)[^3]. |

## References

[^1]: ClickHouse blog / history — open-sourced by Yandex in June 2016. https://clickhouse.com/docs/about-us/history
[^2]: ClickHouse Inc. company/about. https://clickhouse.com/company
[^3]: ClickHouse 26.5 release/community call announcement (README, 2026). https://clickhouse.com/company/events/v26-5-community-release-call

## Tags

database, olap, columnar, analytics, cpp, sql, real-time, dbms, distributed, mergetree, big-data
