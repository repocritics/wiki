# StarRocks/starrocks

> An MPP analytical database (OLAP) with a C++ vectorized engine, MySQL-protocol SQL, and native data-lake query — forked from Apache Doris, now a Linux Foundation project.

[GitHub repo](https://github.com/StarRocks/starrocks) ·
[Official website](https://starrocks.io) ·
[License: Apache-2.0](https://github.com/StarRocks/starrocks/blob/main/LICENSE.txt)

## Overview

StarRocks is a distributed, massively-parallel (MPP) SQL database built for sub-second analytics: multi-dimensional aggregation, real-time upserts, and ad-hoc queries over both its own columnar storage and external data lakes. It began as **DorisDB**, a commercial fork of Apache Doris created by ex-Doris/Baidu engineers (the company now called CelerData), was open-sourced as StarRocks under Apache-2.0 in September 2021, and was contributed to the Linux Foundation in 2023[^1]. The Doris lineage is worth knowing: the two projects share ancestry and a similar FE/BE split, and the fork was not without friction in the upstream community.

The defining design choice is a **two-language split**: the Frontend (FE) is Java — it owns metadata, SQL parsing, the cost-based optimizer, and query coordination; the Backend (BE) is C++ — it owns columnar storage and a hand-written vectorized execution engine. This is where StarRocks's performance claims come from (SIMD-friendly column-at-a-time execution, a CBO, runtime filters, colocated joins) and also where its operational weight comes from: you are running and tuning two very different processes.

StarRocks positions itself against both the "load-and-denormalize" OLAP databases (ClickHouse, Druid, Pinot) and the "query-the-lake" engines (Trino, Presto). Its pitch is that you can do fast **joins** on normalized star-schema data instead of pre-flattening everything, and that the same engine queries Iceberg/Hudi/Delta/Hive tables in place via external catalogs[^2]. Whether that convenience is worth the operational surface is the central tradeoff.

## Getting Started

The fastest path is the all-in-one Docker image (FE + BE in one container, for evaluation only):

```bash
docker run -p 9030:9030 -p 8030:8030 -p 8040:8040 \
  -itd --name quickstart starrocks/allin1-ubuntu
```

Connect with any MySQL client (StarRocks speaks the MySQL wire protocol on port 9030):

```sql
-- mysql -h 127.0.0.1 -P 9030 -u root
CREATE DATABASE demo;
USE demo;

CREATE TABLE events (
  event_day  DATE,
  user_id    BIGINT,
  event_type VARCHAR(32),
  amount     DECIMAL(10,2)
)
DUPLICATE KEY(event_day, user_id)
PARTITION BY date_trunc('day', event_day)
DISTRIBUTED BY HASH(user_id) BUCKETS 8;

INSERT INTO events VALUES ('2026-07-01', 42, 'purchase', 19.99);

SELECT event_type, sum(amount)
FROM events
GROUP BY event_type;
```

For production, FE and BE run as separate clusters; see the manual deployment and Docker-build docs[^3].

## Architecture / How It Works

Two node types form the core:

- **Frontend (FE, Java)** — stores all metadata (schemas, tablet locations, transactions) and runs the SQL layer: parser, analyzer, cost-based optimizer, and coordinator. FE nodes run in **Leader / Follower / Observer** roles; metadata is journaled and replicated via BerkeleyDB JE, with a Leader elected among Followers. Followers provide HA; Observers scale out read/metadata traffic without voting.
- **Backend (BE, C++)** — stores column data as **tablets** (the unit of replication and scheduling) and executes query fragments with the vectorized engine. In the classic shared-nothing deployment BE holds data on local disk with 3× replication.

Data is organized by **partition** (usually by date) then **bucket** (hash distribution → tablets). Getting bucket/tablet counts right is a first-order tuning decision, not an afterthought.

**Table models** determine update semantics — this is a StarRocks-specific concept:
- *Duplicate Key* — append-only, keeps all rows (log/event data).
- *Aggregate Key* — pre-aggregates on load (SUM/MAX/…); good for rollup metrics.
- *Unique Key* — upsert by key via merge-on-read.
- *Primary Key* — upsert/delete with a delete+insert (delete-vector) design, giving fast point-updates and clean reads at the cost of an **in-memory primary-key index**.

Since **version 3.0 (2023)** StarRocks added a **shared-data architecture**: BE is replaced by stateless **Compute Nodes (CN)**, and data lives in object storage (S3/HDFS/etc.) with a local disk cache. This decouples compute from storage and lowers cost for elastic workloads, at the price of cache-warmth sensitivity and a different operational model from shared-nothing[^4].

Other internals: a **pipeline execution engine** for intra-node parallelism; **materialized views** (synchronous single-table rollups and asynchronous multi-table MVs that the optimizer can transparently rewrite queries onto); **external catalogs** for Iceberg/Hudi/Delta/Hive/JDBC; and ingestion paths via Stream Load, Broker Load, **Routine Load** (continuous Kafka consumption), and the Flink/Spark connectors.

## Production Notes

**FE memory and metadata.** The FE keeps metadata (including tablet metadata) on the JVM heap. Clusters with very large numbers of tablets/partitions can pressure FE heap and slow metadata operations. Control tablet proliferation: avoid over-bucketing, prune old partitions, and prefer fewer/larger tablets over many tiny ones.

**BE memory is the common outage.** The C++ BE executes joins/aggregations in memory and will hit its `mem_limit`; large or badly-planned queries surface as query-level OOM or, worse, BE crashes. Watch spill-to-disk settings, query-level memory limits, and resource groups for tenant isolation.

**Primary Key tables cost RAM.** The Primary Key model keeps its index resident in memory for fast upserts. On high-cardinality keys this is a real footgun — size it before adopting Primary Key everywhere just for the upsert convenience. Persistent-index options exist but change the tradeoff.

**Compaction under real-time ingest.** Frequent small loads (Routine Load, high-frequency INSERTs) generate many versions/rowsets that background compaction must merge. If ingest outruns compaction you get read amplification, growing version counts, and "too many versions" load failures. Batch writes and tune compaction threads.

**Upgrades and version skew.** FE and BE must be upgrade-compatible; the supported path is rolling upgrade FE-then-BE within adjacent versions. Metadata is generally **not safely downgradable** once migrated — test upgrades on a staging cluster and keep FE metadata backups. Track the LTS lines (e.g. 2.5, 3.1, 3.2, 3.3) rather than every minor if you value stability.

**Data-lake reads are not free.** External-catalog query performance depends heavily on the data-lake cache, file/manifest layout, and statistics. Cold Iceberg/Hudi scans over object storage can be far slower than native tables; the local **data cache** must warm up before you see the advertised numbers.

## When to Use / When Not

**Use when:**
- You need fast **joins** on normalized/star-schema data and want to avoid denormalizing everything into wide tables.
- You want real-time upserts (Primary Key model) plus low-latency analytical reads in one system.
- You want one MySQL-protocol engine that queries both native tables and Iceberg/Hudi/Delta/Hive lakes.
- You want to decouple compute from storage (3.0+ shared-data) for elastic, cost-sensitive workloads.

**Avoid when:**
- Your workload is single wide-table scans with few joins — ClickHouse is often simpler and competitive there.
- You need ultra-low-latency, high-QPS user-facing lookups — Pinot/Druid are purpose-built for that shape.
- You want a storage-less federation layer over many sources — Trino/Presto fit better; StarRocks is a database, not just a query engine.
- Your team can't own a two-process (Java FE + C++ BE) distributed system — the operational surface is non-trivial.

## Alternatives

- apache/doris — the upstream StarRocks forked from; use Doris when you want the original project's lineage, governance, and community.
- ClickHouse/ClickHouse — use when your workload is denormalized wide-table scans and you want a simpler single-engine story rather than fast star-schema joins.
- apache/pinot — use for ultra-low-latency, high-concurrency user-facing analytics on real-time streams.
- apache/druid — use for time-series and real-time event analytics with a mature ingestion/rollup model.
- trinodb/trino — use when you need federated SQL across many sources with no native storage tier of its own.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2020 | Forked from Apache Doris as DorisDB (commercial). |
| 1.x (OSS) | 2021-09 | Open-sourced as StarRocks under Apache-2.0[^1]. |
| 2.x | 2022 | CBO maturation, Primary Key model, pipeline engine, 2.5 LTS. |
| 3.0 | 2023 | Shared-data architecture (compute/storage separation)[^4]. |
| 3.1 | 2023 | Shared-data GA improvements, more lake-format support. |
| 3.2 | 2024 | Data-lake query and connector improvements. |
| 3.3 | 2024 | LTS line; query and shared-data enhancements. |

## References

[^1]: StarRocks — "About / project history" and Linux Foundation contribution. https://www.starrocks.io/ and https://starrocks.io/blog
[^2]: StarRocks docs — "Catalog / query external data (Iceberg, Hudi, Delta Lake, Hive)". https://docs.starrocks.io/docs/data_source/catalog/catalog_overview/
[^3]: StarRocks docs — "Deploy StarRocks" and "Build in Docker". https://docs.starrocks.io/docs/deployment/deployment_overview/
[^4]: StarRocks docs — "Shared-data architecture / deploy shared-data cluster". https://docs.starrocks.io/docs/deployment/shared_data/s3/

## Tags

olap, mpp, analytics, sql, columnar-database, vectorized, data-lakehouse, iceberg, real-time-analytics, cloud-native, java, cpp
