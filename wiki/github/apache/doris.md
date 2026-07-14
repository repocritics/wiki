# apache/doris

> An MPP analytical (OLAP) database with a MySQL-compatible SQL front end, split into a Java query layer and a C++ storage/execution layer.

[GitHub repo](https://github.com/apache/doris) ·
[Official website](https://doris.apache.org) ·
[License: Apache-2.0](https://github.com/apache/doris/blob/master/LICENSE.txt)

## Overview

Apache Doris is a massively-parallel (MPP) columnar analytics database. It began as **Palo**, an internal OLAP system at Baidu, was open-sourced around 2017, entered the Apache Incubator in 2018, and graduated to a Top-Level Project in June 2022[^1]. Its enduring selling point is that it speaks the MySQL wire protocol and dialect, so existing BI tools and MySQL clients connect with no driver changes, while the engine underneath is a distributed column store built for sub-second aggregation over large tables.

The system's defining split is **two languages, two processes**: the Frontend (FE) is Java and owns metadata, SQL parsing, planning, and cluster coordination; the Backend (BE) is C++ and owns columnar storage, vectorized execution, and I/O[^2]. This gives Doris a Java-ecosystem control plane (easy to extend catalogs, connectors, optimizer) over a C++ data plane (tight memory control, SIMD). The cost is an internal RPC/serialization boundary that shapes every performance discussion and complicates debugging across the two tiers.

Since 2023 Doris has pushed in three directions the marketing now leads with: lakehouse query acceleration over Iceberg/Hudi/Delta via external catalogs, an inverted-index-based full-text/log search path, and — most recently — vector and "hybrid search" features aimed at AI/agent retrieval workloads[^3]. The GitHub description ("real-time analytics and hybrid search database for AI agents") reflects that latest repositioning; the mature, load-bearing core is still relational OLAP.

## Getting Started

Fastest path is the all-in-one Docker image or the release tarball. Doris requires a Frontend and at least one Backend running.

```bash
# Pull and run a single-node all-in-one container (FE + BE)
docker run -itd --name doris-quickstart -p 8030:8030 -p 9030:9030 \
  apache/doris:doris-all-in-one-2.1.0
```

Connect with any MySQL client on port 9030:

```sql
-- Standard MySQL protocol; no Doris-specific driver needed
CREATE DATABASE demo;
USE demo;

CREATE TABLE sales (
    dt      DATE,
    region  VARCHAR(32),
    amount  DECIMAL(18,2)
)
DUPLICATE KEY(dt, region)
DISTRIBUTED BY HASH(region) BUCKETS 10
PROPERTIES ("replication_num" = "1");

INSERT INTO sales VALUES ('2026-01-01', 'kr', 100.00);
SELECT region, sum(amount) FROM sales GROUP BY region;
```

For real ingestion, Stream Load (HTTP `PUT`), Routine Load (Kafka), and the Flink/Spark connectors are the production paths rather than `INSERT`.

## Architecture / How It Works

**Two node types.**
- **FE (Frontend, Java)** — parses SQL, plans and optimizes queries, stores all metadata, and coordinates the cluster. FE nodes elect a Master and replicate metadata via an embedded BerkeleyDB Java Edition (bdbje) journal; Follower/Observer FEs provide HA and read scale-out. FE is the metadata bottleneck: it is not horizontally shardable, only replicated.
- **BE (Backend, C++)** — stores the columnar data (segment files), executes the vectorized plan fragments, and does the actual scanning, joining, and aggregating. BEs scale horizontally and hold the data.

**Storage models.** A table declares one of three key models, and this choice is effectively permanent for the table:
- **Duplicate** — append-only, no dedup; fastest ingest, for logs/events.
- **Aggregate** — pre-aggregates rows sharing the key at load/compaction time (SUM/MAX/REPLACE); trades flexibility for query speed.
- **Unique** — primary-key semantics with upsert. Modern Doris implements this via **Merge-on-Write** (default since 2.0), which pushes dedup cost to write time so reads stay fast — a major reversal of the older Merge-on-Read behavior that made point reads slow[^4].

**Execution.** Since 1.2 the vectorized engine is the default; execution operates on column batches, not row-at-a-time. Doris 2.1 shipped a new cost-based optimizer (Nereids) as the default planner, replacing the legacy planner[^3]. Data is hash-distributed into **buckets** within **partitions**; picking bucket count and partition keys is the single most consequential physical-design decision.

**Compute-storage decoupling.** Classic Doris co-locates compute and storage on BEs. Doris 3.0 added a decoupled mode where stateless compute groups run over shared object storage (S3/HDFS), enabling elastic compute and workload isolation[^5]. The two deployment modes are a fork in operational model, not a runtime toggle — you choose at cluster build time.

**Lakehouse.** Multi-Catalog lets Doris mount external metastores (Hive, Iceberg, Hudi, Delta, JDBC sources) and query them in place, using Doris as an acceleration/federation engine rather than the system of record.

## Production Notes

- **FE is the metadata single-writer.** All DDL and metadata mutations serialize through the FE Master. Very high table/partition/tablet counts inflate FE memory and metadata-replay time; clusters with hundreds of thousands of tablets hit FE GC pauses and slow restarts. Tablet count management (bucket sizing, partition granularity) is the recurring operational headache.
- **Bucketing mistakes are expensive.** Too few buckets underutilize BEs; too many create tiny tablets that bloat FE metadata and compaction load. Bucket count is set at create time and changing it means rebuilding the table (or using AUTO buckets on newer versions, which mitigates but does not eliminate the concern).
- **Compaction pressure.** High-frequency small loads (many small Stream Loads) generate many rowsets that background compaction must merge. Under sustained high ingest, compaction can fall behind, degrading query performance and disk usage — batch loads and tune compaction threads.
- **Memory model.** BE is C++ with its own memory tracking; OOM on BE kills queries or the node. Query memory limits, and the historical lack of robust spill-to-disk (improving in recent versions), mean large joins/aggregations can fail rather than degrade gracefully.
- **Two-tier debugging.** A slow or failing query may be an FE planning issue or a BE execution issue; you need to read both `fe.log` and `be.INFO`/`be.WARNING`, and understand profiles that span the Java→C++ boundary.
- **Upgrades.** Doris generally supports rolling upgrades BE-first, but cross-major upgrades (1.x→2.x, 2.x→3.x) have introduced metadata format and default-behavior changes (e.g., Merge-on-Write and Nereids becoming defaults) that warrant staging and rollback planning. Read the version-specific upgrade notes; do not skip majors blindly.
- **Newer features are newer.** Inverted-index search, the variant/JSON type, and vector/hybrid search are comparatively young relative to the relational core; validate them against your workload before betting production on them.

## When to Use / When Not

**Use when:**
- You want fast interactive/aggregation SQL over large tables with MySQL-protocol compatibility and no separate serving layer.
- You need customer-facing analytics or a real-time warehouse with high-concurrency, sub-second queries.
- You want one engine that both stores hot data and federates/accelerates a lakehouse (Iceberg/Hudi/Delta).
- You value a single SQL surface over structured + JSON + text (+ increasingly vector) data.

**Avoid when:**
- You need OLTP: Doris is analytical: high-frequency small point writes and transactional updates are not its shape.
- Your workload is single-node or embedded: DuckDB/ClickHouse-single-node is simpler with no FE/BE cluster to run.
- You have thin ops capacity and cannot manage bucketing, compaction, and FE/BE tuning: the defaults are reasonable but production scale demands hands-on tuning.
- Your primary need is a mature dedicated vector database: use a purpose-built one; Doris vector search is a recent add-on.

## Alternatives

- StarRocks/starrocks — a fork of the Doris codebase by a former Baidu team; use it instead when you want the same MySQL-protocol MPP OLAP model but prefer that project's optimizer/lakehouse trajectory.
- ClickHouse/ClickHouse — use instead when raw single-table columnar scan throughput matters more than joins, upserts, and cluster ergonomics.
- apache/pinot — use instead for ultra-low-latency user-facing analytics with heavy pre-aggregation and high-ingest upserts.
- apache/druid — use instead for time-series and event/streaming analytics with real-time ingestion as the first-class case.
- duckdb/duckdb — use instead when you want embedded, single-process analytics with no distributed cluster at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| (Palo) | ~2017 | Open-sourced from Baidu's internal Palo OLAP system[^1]. |
| Incubator | 2018 | Entered the Apache Incubator. |
| TLP | 2022-06 | Graduated to Apache Top-Level Project[^1]. |
| 1.2 | 2022-12 | Vectorized engine default; Multi-Catalog for lakehouse federation. |
| 2.0 | 2023-08 | Inverted index / full-text search; Merge-on-Write default for Unique[^4]. |
| 2.1 | 2024-03 | Nereids cost-based optimizer as default; variant (semi-structured) type[^3]. |
| 3.0 | 2024 | Compute-storage decoupled deployment mode[^5]. |

## References

[^1]: Apache Software Foundation, "Apache Doris" project page and incubation/graduation record. https://doris.apache.org/ and https://incubator.apache.org/projects/doris.html
[^2]: Apache Doris docs, "Introduction to Apache Doris / Technical Overview" (FE Java + BE C++ architecture). https://doris.apache.org/docs/dev/getting-started/what-is-apache-doris
[^3]: Apache Doris release notes (2.x series — Nereids optimizer, variant type, search features). https://doris.apache.org/releases/all-release
[^4]: Apache Doris docs, "Data Model" and 2.0 release notes (Merge-on-Write for the Unique key model). https://doris.apache.org/docs/dev/table-design/data-model/overview
[^5]: Apache Doris docs, "Choosing a deployment mode" (compute-storage coupled vs decoupled). https://doris.apache.org/docs/dev/install/choosing-deployment-mode

## Tags

olap, mpp, columnar-database, sql, analytics, lakehouse, real-time, java, cpp, mysql-protocol, data-warehouse, apache
