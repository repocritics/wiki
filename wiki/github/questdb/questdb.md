# questdb/questdb

> Column-oriented, SQL-native time-series database tuned for high-throughput ingestion and fast range/aggregation queries — strongest on financial tick data.

[GitHub repo](https://github.com/questdb/questdb) ·
[Official website](https://questdb.com) ·
[License: Apache-2.0](https://github.com/questdb/questdb/blob/master/LICENSE.txt)

## Overview

QuestDB is a time-series database written primarily in low-level, zero-GC Java plus C++ (with some Rust in the Enterprise build)[^1]. The public repo dates to 2014 as a personal project by founder Vlad Ilyushchenko; it was open-sourced and later commercialized through Y Combinator's 2020 batch. It speaks SQL with time-series extensions, ingests over the InfluxDB Line Protocol (ILP) and PostgreSQL wire protocol, and stores data as time-partitioned columnar files. At ~17.2k stars it is one of the more visible open-source TSDBs, and the one most explicitly optimized for capital-markets workloads (trades, order books, OHLC)[^2].

The defining tradeoff is specialization. QuestDB is very fast at the shape of query it was built for — filter by time range, aggregate, downsample, ASOF-join two ordered streams — and comparatively thin everywhere else. It is not a general relational database: DELETE of arbitrary rows is unsupported, joins beyond time-oriented ones are less optimized, and the SQL surface, while growing, is a subset of what Postgres or ClickHouse expose. You adopt QuestDB when time is the primary axis and ingestion rate matters more than relational generality.

The other structural fact to internalize early: high availability, replication, and role-based access control live only in QuestDB Enterprise. The open-source build is single-node[^3].

## Getting Started

```bash
# Docker — 9000 HTTP/REST/Web Console/ILP-over-HTTP, 8812 PGWire, 9009 ILP/TCP
docker run -p 9000:9000 -p 8812:8812 -p 9009:9009 questdb/questdb

# or macOS via Homebrew
brew install questdb && questdb start
```

```sql
-- Web Console (http://localhost:9000) or any Postgres client on :8812
CREATE TABLE trades (
  ts        TIMESTAMP,
  symbol    SYMBOL,          -- interned low-cardinality string
  price     DOUBLE,
  amount    DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY WAL;

-- downsample to 1-minute OHLC buckets
SELECT ts, symbol,
       first(price) o, max(price) h, min(price) l, last(price) c
FROM trades
WHERE ts IN '2026-07-14'
SAMPLE BY 1m;
```

## Architecture / How It Works

**Storage.** Each table is a set of append-only column files on disk, partitioned by a designated `TIMESTAMP` column (by `HOUR`/`DAY`/`MONTH`/`YEAR`). Columns are stored contiguously, so range scans read only the columns and partitions a query touches. Data is expected to arrive roughly time-ordered; genuinely out-of-order (O3) rows trigger a partition rewrite/merge rather than an in-place append.

**Ingestion path.** Modern tables are **WAL tables**: writes land in a write-ahead log and are applied by a background job, which decouples ingestion from readers, allows multiple concurrent writers, and enables commit-time deduplication[^4]. Non-WAL tables (the older single-writer model) still exist but are legacy. ILP is the high-throughput ingest interface; PGWire and the REST API are the query interfaces.

**Query engine.** SQL is compiled to a vectorized execution plan using SIMD-accelerated operators and parallelized across cores. Time-series-specific operators — `SAMPLE BY` (time bucketing), `LATEST ON` (last value per series), `ASOF JOIN` / `LT JOIN` (join by nearest preceding timestamp) — are the reason to choose QuestDB over a generic column store, and they depend on the designated timestamp being present and ordered.

**Types.** The `SYMBOL` type interns repeated strings into an integer dictionary; it is the right choice for low-to-moderate-cardinality tags (instrument, exchange) and the wrong choice for unbounded/high-cardinality values, where the dictionary bloats. Newer versions add n-dimensional arrays (e.g. 2D arrays for order-book levels) and a Parquet storage tier for colder partitions.

Most data lives off-heap in memory-mapped files, so resident memory tracks the OS page cache rather than a JVM heap — expect large RSS numbers that are not a leak.

## Production Notes

- **Single node in OSS.** No replication, failover, or HA without Enterprise. Durability is local disk plus the WAL; treat a QuestDB instance as a single point of failure and plan backups (`SNAPSHOT` / filesystem snapshot) accordingly[^3].
- **Out-of-order write amplification.** Ingesting rows far behind the current time forces QuestDB to rewrite the affected partition. Steady near-real-time streams are cheap; large late backfills into old partitions are expensive and can stall ingestion. Batch historical loads partition-aligned.
- **Deletes are partition-granular.** There is no arbitrary row `DELETE`. You reclaim data with `ALTER TABLE ... DROP PARTITION`. `UPDATE` exists but is not the fast path — design for append-only.
- **SYMBOL capacity matters.** Symbol columns have configured capacity; undersizing hurts ingestion, and using SYMBOL for high-cardinality data (e.g. unique IDs) degrades both memory and speed. Use `VARCHAR`/`STRING` there instead.
- **SQL is a subset.** Correlated subqueries, some window functions, and general non-time joins are limited or slower than a mature RDBMS. Validate that your analytical queries express cleanly before committing.
- **Timestamp is load-bearing.** Time-series features silently do nothing useful without a designated, ordered `TIMESTAMP`. `SAMPLE BY`, `LATEST ON`, and ASOF joins all assume it.
- **Cardinality model.** QuestDB is built for wide, deep time-series, not for Prometheus-style explosion of label combinations. It is not a drop-in metrics/monitoring backend.

## When to Use / When Not

**Use when:**
- Time is the primary query axis: tick data, trades/order books, sensor and telemetry streams, real-time dashboards.
- You need sustained high-rate ingestion with concurrent low-latency reads on a single beefy node.
- You want SQL plus `SAMPLE BY` / `ASOF JOIN` rather than a bespoke query DSL.

**Avoid when:**
- You need built-in HA/replication without paying for Enterprise.
- Your workload is relational (frequent updates/deletes, many-table joins, transactions) — reach for Postgres/TimescaleDB.
- Your data is high-cardinality metrics with label-set fan-out — a Prometheus-model store fits better.
- You need a general OLAP warehouse over non-time-keyed data — ClickHouse is broader.

## Alternatives

- influxdata/influxdb — use when you want the broadest TSDB ecosystem and managed cloud; InfluxDB 3 rebuilt on Rust/Arrow/Parquet.
- timescale/timescaledb — use when you want full PostgreSQL SQL, joins, and transactions on time-series as a Postgres extension.
- clickhouse/clickhouse — use when you need general-purpose columnar OLAP at cluster scale beyond a time axis.
- VictoriaMetrics/VictoriaMetrics — use for Prometheus-compatible, label-based metrics and monitoring at scale.
- apache/druid — use for high-concurrency interactive OLAP dashboards over a distributed cluster.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2014-04 | Repo created as founder's personal project[^1]. |
| YC S20 | 2020 | Commercialized via Y Combinator; open-source core on Apache-2.0. |
| 6.0 | 2021 | Out-of-order (O3) ingestion — accept unordered timestamps. |
| 7.0 | 2023 | WAL tables: concurrent writers + commit-time deduplication[^4]. |
| 8.x | 2024–2026 | Parquet storage tier, n-dimensional arrays, materialized views. |

## References

[^1]: QuestDB repository metadata (created 2014-04-28) and project background. https://github.com/questdb/questdb
[^2]: QuestDB README — feature highlights and target workloads (finance/tick data, sensor/telemetry, real-time dashboards). https://github.com/questdb/questdb/blob/master/README.md
[^3]: QuestDB Enterprise — HA, read replicas, RBAC, TLS are Enterprise-only. https://questdb.com/enterprise/
[^4]: QuestDB docs — write-ahead log (WAL) tables and deduplication. https://questdb.com/docs/concept/write-ahead-log/

## Tags

time-series-database, java, cpp, sql, columnar, olap, ingestion, market-data, simd, apache-2.0, tsdb, real-time-analytics
