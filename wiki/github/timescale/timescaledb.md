# timescale/timescaledb

> A time-series and real-time-analytics engine shipped as a PostgreSQL extension, not a separate database.

[GitHub repo](https://github.com/timescale/timescaledb) ·
[Official website](https://www.tigerdata.com) ·
[License: Apache-2.0 + Timescale License (TSL)](https://github.com/timescale/timescaledb/blob/main/LICENSE)

## Overview

TimescaleDB is a PostgreSQL extension (written in C) that adds automatic time-based partitioning, columnar compression, and incrementally-maintained aggregates to a stock Postgres server[^1]. Its central bet is architectural: rather than build a new storage engine, it stays inside Postgres and inherits the full SQL surface, the planner, the ecosystem (drivers, `pg_dump`, replication, extensions like PostGIS and pgvector), and the operational knowledge teams already have. You install it with `CREATE EXTENSION timescaledb` and keep writing SQL.

The defining abstraction is the **hypertable**: a table that looks like one relation but is transparently split into many child tables ("chunks") partitioned by a time column. Queries and inserts address the hypertable; the extension routes them to the right chunks and prunes irrelevant ones. On top of hypertables sit columnar compression (the "columnstore"), continuous aggregates (materialized views that refresh only changed ranges), retention policies, and `time_bucket`-family hyperfunctions.

The project is developed by Timescale, Inc., which rebranded to **Tiger Data** in 2026 — the homepage, docs, and managed offering (Tiger Cloud, formerly Timescale Cloud) now carry that name, though the GitHub org, extension name, and `timescaledb` package are unchanged[^2]. The defining tension is licensing: the core is Apache-2.0, but most of the features people adopt TimescaleDB *for* (compression, continuous aggregates, data tiering, several policies) ship under the source-available **Timescale License (TSL)**, which forbids offering them as a competing managed service[^3]. That is also why you cannot get TimescaleDB on AWS RDS or most managed-Postgres providers.

## Getting Started

TimescaleDB is loaded via `shared_preload_libraries`; the Docker image handles that for you:

```bash
docker run -d --name timescaledb -p 5432:5432 \
    -e POSTGRES_PASSWORD=password \
    timescale/timescaledb-ha:pg18
```

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- A hypertable: ordinary table + time partitioning
CREATE TABLE sensor_data (
    time        TIMESTAMPTZ NOT NULL,
    sensor_id   TEXT        NOT NULL,
    temperature DOUBLE PRECISION,
    humidity    DOUBLE PRECISION
) WITH (tsdb.hypertable);   -- older syntax: SELECT create_hypertable('sensor_data','time');

INSERT INTO sensor_data
SELECT t, 'sensor_' || (random()*9)::int, 20 + random()*15, 40 + random()*30
FROM generate_series(now() - interval '90 days', now(), interval '1 second') AS t;

-- time_bucket: the workhorse aggregation primitive
SELECT time_bucket('1 hour', time) AS hour,
       sensor_id, avg(temperature)
FROM sensor_data
WHERE time > now() - interval '24 hours'
GROUP BY hour, sensor_id;
```

## Architecture / How It Works

**Hypertables and chunks.** A hypertable is a virtual parent; each chunk is a real Postgres table covering one time interval (`chunk_time_interval`, default 7 days). Inserts are routed to the chunk owning their timestamp; queries use *chunk exclusion* to skip chunks outside the `WHERE` time range before planning even touches them. Optional space partitioning (by a second key) shards further. Everything is regular Postgres underneath, so indexes, constraints, triggers, and `EXPLAIN` behave normally per chunk.

**Columnstore (compression).** Chunks can be converted from row storage to a columnar format that groups rows into compressed batches, chosen per-column (delta-of-delta, Gorilla, dictionary, LZ, etc.)[^4]. Compression is configured with `segmentby` (columns used for grouping/filtering) and `orderby` (sort within a batch). Reported ratios of 90%+ are common on well-ordered numeric series. Columnar chunks also enable vectorized execution for aggregates.

**Continuous aggregates.** These are materialized views declared `WITH (timescaledb.continuous)`. Unlike a plain Postgres materialized view (full rebuild on refresh), a continuous aggregate tracks an invalidation log and re-materializes only the time ranges that changed. A "real-time aggregation" mode unions the materialized result with freshly-arrived raw rows so queries stay current between refreshes.

**Background jobs.** Retention, compression, and continuous-aggregate refresh are all driven by policies run by a background-worker scheduler inside Postgres. `add_retention_policy`, `add_columnstore_policy`, and `add_continuous_aggregate_policy` schedule them; the jobs show up in `timescaledb_information.jobs`.

**Data tiering.** On Tiger Cloud, older chunks can be moved to S3-backed object storage and still queried transparently. This is a managed-service feature, not part of the open extension.

## Production Notes

**RDS and managed Postgres are out.** Because TimescaleDB needs `shared_preload_libraries` and because the TSL features can't be offered by a competing host, AWS RDS, Aurora, GCP Cloud SQL, and Azure Database do **not** offer it. Your options are self-hosting, the vendor's Tiger Cloud, or the small set of hosts (e.g. some managed-Postgres vendors) that struck agreements. Plan hosting before you build on TSL features.

**Multi-node is gone.** TimescaleDB once shipped a distributed/multi-node scale-out mode; it was deprecated and then removed (the extension is single-node again), with the scaling story now compression + tiering rather than horizontal sharding[^5]. Do not architect around distributed hypertables on current versions.

**Chunk sizing is a real tuning knob.** Too-large chunks defeat exclusion and blow past memory during compression; too-small chunks create thousands of tables and planner overhead. The rule of thumb is that a chunk's indexes for the most recent interval should fit in memory. Re-chunking existing data is not automatic.

**Compression changes write semantics.** Historically, once a chunk was compressed it was effectively read-mostly — `UPDATE`/`DELETE` and out-of-order inserts into compressed chunks were restricted or slow, requiring decompression. Later versions relaxed this considerably, but if you backfill or mutate old data, test the write path on your actual version before committing to a compression policy.

**Cardinality bites compression and continuous aggregates.** A high-cardinality `segmentby` column produces many tiny batches and poor ratios; wide `GROUP BY` in continuous aggregates inflates the materialization. Model these deliberately rather than by default.

**Upgrades.** Extension upgrades are `ALTER EXTENSION timescaledb UPDATE` in a fresh connection (the update can't run in a session that already loaded the old version), and must be paired with a matching binary/image. Cross-major-Postgres moves (e.g. pg16 → pg18) follow standard Postgres `pg_upgrade` plus an extension update; read the release notes, as some releases changed catalog internals.

## When to Use / When Not

**Use when:**
- You already run Postgres and want time-series scale without migrating to a separate system.
- Your workload is append-heavy time-stamped data (metrics, IoT, events, financial ticks) queried with time-bucketed aggregations.
- You need relational joins, full SQL, and existing Postgres tooling alongside time-series features.
- Compression of large historical ranges plus cheap rollups (continuous aggregates) is the shape of your problem.

**Avoid when:**
- You need a fully OSI-open stack or intend to resell it as a service — the best features are TSL, not open source.
- You need managed Postgres on RDS/Aurora/Cloud SQL specifically.
- Your data isn't really time-series (no natural time-ordering to partition on); you get overhead without the wins.
- You need horizontal write sharding across nodes — that path was removed.
- Your ingest is extreme-cardinality metrics at monitoring scale where a purpose-built TSDB (Prometheus/VictoriaMetrics) or columnar OLAP store fits better.

## Alternatives

- influxdata/influxdb — purpose-built TSDB; InfluxDB 3.x is columnar (Rust/Arrow/Parquet). Use it when you want a dedicated time-series system and don't need relational SQL/joins.
- questdb/questdb — high-ingest columnar TSDB with SQL. Use it when raw ingest throughput and simple time-series analytics matter more than the Postgres ecosystem.
- ClickHouse/ClickHouse — columnar OLAP. Use it when analytical scan performance over huge datasets outweighs transactional/relational needs and per-row updates.
- VictoriaMetrics/VictoriaMetrics or prometheus/prometheus — use for monitoring/metrics with label-based cardinality and PromQL rather than SQL.
- Plain postgres/postgres with native partitioning + BRIN indexes — use when your time-series volume is modest and you'd rather not take an extension dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017-04 | First public release; hypertables on Postgres, Apache-2.0[^1]. |
| 1.0 | 2018-10 | Production GA; Timescale License (TSL) introduced for enterprise features[^3]. |
| 2.0 | 2021-01 | Continuous aggregates redesign, native compression GA, multi-node (later removed)[^6]. |
| 2.x | 2022–2026 | Compression/columnstore improvements, vectorized aggregation, hyperfunctions, data tiering; multi-node deprecated then removed. |
| — | 2026 | Company rebrands Timescale → Tiger Data; PostgreSQL 18 support[^2]. |

## References

[^1]: TimescaleDB README and repository (org `timescale`, extension in C). https://github.com/timescale/timescaledb
[^2]: Tiger Data (formerly Timescale) — product and documentation site. https://www.tigerdata.com / https://docs.tigerdata.com
[^3]: Timescale License (TSL) — source-available license covering community/enterprise features atop the Apache-2.0 core. https://github.com/timescale/timescaledb/blob/main/tsl/LICENSE-TIMESCALE
[^4]: TimescaleDB compression / columnstore documentation. https://docs.tigerdata.com/use-timescale/latest/compression/about-compression/
[^5]: Multi-node deprecation/removal in TimescaleDB release notes. https://github.com/timescale/timescaledb/blob/main/CHANGELOG.md
[^6]: TimescaleDB 2.0 announcement (continuous aggregates, compression, multi-node). https://www.tigerdata.com/blog

## Tags

c, postgresql, postgres-extension, time-series, database, tsdb, real-time-analytics, columnar-storage, iot, sql, olap
