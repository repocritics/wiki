# apache/hive

> SQL over petabyte-scale data in Hadoop storage — and the source of the Metastore that became the industry's default table catalog.

[GitHub repo](https://github.com/apache/hive) ·
[Official website](https://hive.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/hive/blob/master/LICENSE)

## Overview

Apache Hive is a data-warehouse layer that exposes SQL (HiveQL) over data sitting in distributed storage — originally HDFS, now just as often S3, ADLS, or GCS. It was built at Facebook around 2007–2008 to let analysts query Hadoop without writing MapReduce jobs, was contributed to the Apache Software Foundation, and became a top-level project in 2010[^1]. For most of the early 2010s "Hadoop SQL" and "Hive" were effectively synonyms.

Hive translates a SQL query into a DAG of distributed tasks and runs it on an execution engine. It is explicitly a batch/warehouse system: schema-on-read, high throughput, high per-query latency. It is not an OLTP database and not a low-latency serving layer — point lookups and single-row updates are the wrong job for it, even with the ACID features added later.

The defining fact about Hive in 2026 is the split between its two halves. The **query engine** has been steadily outrun by Spark SQL, Trino, and Impala on both latency and adoption. The **Hive Metastore (HMS)** — the RDBMS-backed service that stores table schemas, partitions, and storage locations — quietly became the de facto metadata catalog for the whole Hadoop/lakehouse ecosystem: Spark, Trino, Presto, Flink, and Impala all read it. Hive's most durable legacy is a catalog, not a compiler. And that catalog is now itself being displaced by Iceberg REST catalogs, Unity Catalog, and Polaris — so both halves are under pressure.

## Getting Started

Hive is not a `brew install` product; it is a service stack that expects a Hadoop-compatible filesystem and a relational database for the Metastore. A minimal local bring-up (Hive 4.x):

```bash
# Requires Java (8 for 4.0.x, 17 for 4.1.x, 21 for 4.2.x) and Hadoop 3.x on PATH
export HIVE_HOME=/opt/apache-hive-4.0.1-bin
export PATH=$HIVE_HOME/bin:$PATH

# Initialize the Metastore schema (Derby here; use MySQL/Postgres in prod)
schematool -dbType derby -initSchema

# Start the metastore + HiveServer2, then connect with Beeline (JDBC)
hive --service metastore &
hive --service hiveserver2 &
beeline -u "jdbc:hive2://localhost:10000"
```

```sql
-- Partitioned, columnar (ORC) external table over cloud storage
CREATE EXTERNAL TABLE events (
  user_id BIGINT,
  action  STRING,
  ts      TIMESTAMP
)
PARTITIONED BY (dt STRING)
STORED AS ORC
LOCATION 's3a://my-bucket/warehouse/events/';

-- Partitions are metadata; register new ones after data lands
MSCK REPAIR TABLE events;   -- or: ALTER TABLE events ADD PARTITION (dt='2026-07-14');

SELECT action, COUNT(*) AS n
FROM events
WHERE dt = '2026-07-14'
GROUP BY action
ORDER BY n DESC;
```

## Architecture / How It Works

A query flows through several decoupled components:

1. **HiveServer2 (HS2)** — a Thrift service that accepts JDBC/ODBC connections via the Beeline client (which replaced the legacy `hive` CLI). It handles sessions, authentication, and query compilation.
2. **Driver / compiler** — parses HiveQL, does semantic analysis against the Metastore, and builds a logical plan. Cost-based optimization runs through **Apache Calcite** (join reordering, predicate pushdown) — but the CBO is only as good as the table/column statistics collected with `ANALYZE TABLE`.
3. **Hive Metastore (HMS)** — a separate service backed by a real RDBMS (MySQL, PostgreSQL, Oracle, MSSQL, or Derby for toy setups). It stores databases, tables, columns, partitions, and physical locations. This is the component other engines consume directly.
4. **Execution engine** — the plan compiles to a DAG that runs on **Apache Tez** (the default and intended engine), historically on **MapReduce** (deprecated and removed as a supported engine in Hive 4), and for a period on **Spark**. **LLAP** (Live Long And Process) adds long-lived daemons with in-memory columnar caching for interactive latencies.

Storage is pluggable through SerDes and input/output formats. **ORC** is Hive's native columnar format (predicate pushdown, lightweight indexes, built-in ACID support); Parquet, Avro, and delimited text are also first-class. As of Hive 4, **Apache Iceberg** tables are supported natively, positioning Hive as one query engine over a modern table format rather than the owner of the storage layout.

**ACID** transactions (insert/update/delete, `MERGE`) work by writing base plus delta files and reconciling them at read time; a background **compactor** (minor/major compaction) merges deltas so read amplification does not grow unbounded. This only works on transactional (ORC) tables and is a real operational subsystem, not a free flag.

## Production Notes

**The Metastore is the bottleneck and the blast radius.** Almost every scaling problem eventually traces back to HMS. Tables with tens or hundreds of thousands of partitions turn planning-time metadata calls into slow RDBMS scans; a slow or overloaded Metastore DB degrades every engine pointed at it, not just Hive. Right-size the backing database, connection-pool it, and be conservative with partition cardinality (partition by date, not by a high-cardinality key).

**The small-files problem is real.** Hive is sensitive to file count and size. Thousands of tiny files per partition (a classic output of over-parallelized Spark/streaming writes) cripple scan performance and inflate Metastore/NameNode load. Compaction, the `hive.merge.*` settings, and bucketing are the mitigations.

**Latency is batch-shaped.** Even on Tez, first-query overhead is seconds; Hive is built for scans over large data, not sub-second dashboards. For interactive latency you run **LLAP** (and accept its always-on daemon footprint and memory tuning) or you put Trino/Impala in front of the same Metastore.

**Statistics decide your plans.** The cost-based optimizer is only useful with current table and column stats. Stale or missing stats silently produce bad join orders and skewed shuffles. `ANALYZE TABLE ... COMPUTE STATISTICS` is operational hygiene, not a one-time step.

**ACID adds a compaction operations burden.** Update/delete-heavy tables need the compactor running and tuned, or reads slow down as deltas accumulate. Failed or lagging compactions are a common source of mysterious read regressions.

**Upgrades touch the Metastore schema.** Every major version ships schema changes; you must run `schematool -upgradeSchema` (scripts provided for MySQL, PostgreSQL, Oracle, MSSQL, Derby). Skipping or misordering these is a classic upgrade failure. The Hive-2 → Hive-3 jump also changed ACID defaults and moved managed tables under transactional semantics, which surprised many operators. Hive 4 requires Hadoop 3.x and a JDK that varies by minor version (Java 8 for 4.0.x, 17 for 4.1.x, 21 for 4.2.x).

**The engine story has churned.** MapReduce is gone, Hive-on-Spark faded, Tez is the answer. Deployments carrying old MapReduce-era config or Hive-on-Spark assumptions should not expect them to survive a Hive 4 migration.

## When to Use / When Not

**Use when:**
- You have large existing data in HDFS/S3/ADLS and want SQL over it in a batch ETL / warehouse pattern.
- You already run a Hive Metastore and want another engine to share that catalog.
- You need mature, stable SQL DDL/DML over ORC/Parquet with ACID and Iceberg support in a Hadoop-native stack.
- Throughput on large scans matters more than per-query latency.

**Avoid when:**
- You need interactive/BI latency out of the box — reach for Trino, Impala, or a columnar OLAP store instead (optionally over the same Metastore).
- You want OLTP, point lookups, or high-frequency single-row updates.
- You are greenfield without a Hadoop investment — Spark SQL or Trino over object storage plus a modern catalog is usually a lighter start.
- You want a single binary / low operational surface: Hive is a multi-service system (HS2 + Metastore + engine + a backing RDBMS).

## Alternatives

- apache/spark — use instead when you want one engine for batch, streaming, and ML with in-memory performance; Spark SQL can read the same Hive Metastore.
- trinodb/trino — use instead for interactive, federated, low-latency SQL across many sources; frequently deployed on top of the Hive Metastore/tables.
- apache/impala — use instead for MPP low-latency SQL on the same Hadoop/HMS data when you need consistently interactive response.
- apache/iceberg — use instead (or alongside) when you want a modern table format with snapshots, schema evolution, and hidden partitioning; Hive 4 can query it.
- clickhouse/clickhouse — use instead when you need real-time, high-concurrency analytical serving rather than batch warehouse scans.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2008 | Created at Facebook; contributed to Apache as a Hadoop subproject. |
| TLP | 2010-09 | Promoted to Apache top-level project[^1]. |
| 1.0 | 2015-02 | Renamed from 0.14 to signal API stability. |
| 2.0 | 2016-02 | LLAP introduced; HPL/SQL procedural SQL[^2]. |
| 2.1 | 2016-06 | Cost-based optimizer maturation via Calcite. |
| 3.0 | 2018-05 | ACID v2 default, materialized views, Tez-only execution[^3]. |
| 4.0.0 | 2024-03 | Native Apache Iceberg support, standalone Metastore, MapReduce removed[^4]. |
| 4.0.1 | 2024-09 | Maintenance release on the 4.0 line (Java 8). |

## References

[^1]: Apache Hive project site and history. https://hive.apache.org/
[^2]: Apache Hive LLAP overview. https://cwiki.apache.org/confluence/display/Hive/LLAP
[^3]: Apache Hive administration/configuration (ACID, engines). https://cwiki.apache.org/confluence/display/Hive/AdminManual+Configuration
[^4]: Apache Hive downloads / 4.0.0 release. https://hive.apache.org/general/downloads/

## Tags

sql, data-warehouse, big-data, hadoop, java, hive-metastore, etl, batch-processing, olap, distributed-sql, apache
