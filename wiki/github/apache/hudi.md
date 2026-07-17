# apache/hudi

> A data lakehouse table format built around fast upserts, deletes, and incremental processing on cloud object storage.

[GitHub repo](https://github.com/apache/hudi) ·
[Official website](https://hudi.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/hudi/blob/master/LICENSE)

## Overview

Hudi (originally "Hadoop Upserts Deletes and Incrementals") is a table format
and a set of writer/reader engines that turn a directory of Parquet files on
object storage into a mutable, transactional table[^1]. It was created at Uber
around 2016 to solve a specific problem: their ingestion pipelines needed to
apply row-level updates and deletes to petabyte-scale Hadoop tables and to read
only what changed since the last run, rather than rescanning whole partitions.
It was open-sourced in 2017, entered the Apache Incubator in January 2019, and
became a top-level Apache project in 2020[^2].

Hudi is one of three projects in the "open table format" space, alongside
Apache Iceberg and Delta Lake. Its defining bias is toward write-side mutation
and streaming: fast record-level upserts, change-data-capture-style incremental
queries, and a runtime of automatic "table services" (compaction, clustering,
cleaning) that maintain the table as data lands. That bias is also the source
of its central tension. Hudi ships more moving parts than its peers — indexes,
a timeline, a metadata table, background services with their own scheduling —
and those parts must be understood and tuned. It rewards streaming upsert
workloads and punishes teams that expected a passive file layout.

## Getting Started

Hudi is a library loaded into a compute engine (Spark, Flink, or others), not a
standalone service. The common entry point is Spark with the Hudi bundle jar:

```bash
spark-shell \
  --packages org.apache.hudi:hudi-spark3.5-bundle_2.12:1.0.2 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
  --conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
  --conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog'
```

```python
# Write a Hudi table, then read only the rows that changed since a commit
hudi_opts = {
    "hoodie.table.name": "trips",
    "hoodie.datasource.write.recordkey.field": "uuid",
    "hoodie.datasource.write.precombine.field": "ts",
    "hoodie.datasource.write.table.type": "MERGE_ON_READ",
}
df.write.format("hudi").options(**hudi_opts) \
    .mode("append").save("s3://bucket/trips")

# Incremental query: only records committed after a given instant
spark.read.format("hudi") \
    .option("hoodie.datasource.query.type", "incremental") \
    .option("hoodie.datasource.read.begin.instanttime", "20260101000000") \
    .load("s3://bucket/trips")
```

## Architecture / How It Works

A Hudi table is a set of data files plus a `.hoodie/` directory holding the
**timeline** — an ordered log of atomic actions (commits, cleans, compactions,
rollbacks), each with requested/inflight/completed states. The timeline is the
source of truth for what a reader sees; snapshot isolation comes from readers
pinning the latest completed instant.

Two table types define the write/read tradeoff:

- **Copy-on-Write (CoW).** Updates rewrite the affected Parquet file groups at
  write time. Reads are pure columnar Parquet scans (fast), writes pay the merge
  cost up front (slow, write-amplified).
- **Merge-on-Read (MoR).** Updates are appended to row-oriented delta log files
  and merged with base Parquet at read time. Writes are cheap and low-latency;
  reads pay a merge cost until a background **compaction** folds logs into new
  base files. MoR is the format that makes Hudi attractive for streaming.

Correspondingly there are query types: **snapshot** (latest merged state),
**read-optimized** (base files only, skips un-compacted deltas — faster but
stale), **incremental** (only records changed since an instant), and
**time-travel** (state as of a past instant).

To make upserts fast, Hudi maintains an **index** mapping record keys to file
groups so a writer knows where an incoming key already lives. Options range from
bloom-filter indexes to a metadata-table-backed **record-level index**. The
**metadata table** is itself an internal MoR Hudi table that stores file
listings and column stats to avoid expensive object-store `list` calls and to
enable data skipping. **Table services** — compaction, clustering, cleaning,
indexing — run inline with the writer or as separate async jobs, each with its
own scheduling and failure handling. This machinery is the reason Hudi is
powerful for mutation-heavy pipelines and also the reason it has more surface
area to operate than a format that only tracks file manifests.

## Production Notes

- **MoR compaction is an operational responsibility, not an afterthought.** If
  writers append delta logs faster than compaction retires them, snapshot-query
  latency degrades and file groups bloat. Teams must decide between inline
  compaction (simpler, adds latency to the write job) and async/offline
  compaction (better throughput, one more job to schedule and monitor).
- **Small-file and cleaning management matter.** Hudi sizes files during writes
  and expires old file versions via the cleaner; misconfigured retention either
  breaks in-flight incremental readers/time-travel or lets storage and metadata
  grow unbounded. Cleaner and compaction retention must be reasoned about
  together.
- **The metadata table adds correctness-sensitive state.** It speeds up
  listings and data skipping but is another MoR table to keep consistent;
  historically some concurrency and corruption edge cases have centered on it,
  so multi-writer setups need care.
- **Concurrency.** Hudi defaults to a single writer with async table services.
  Multiple concurrent writers require Optimistic Concurrency Control (OCC) with
  an external lock provider (ZooKeeper, DynamoDB, Hive Metastore); OCC aborts
  losers on conflict, so contended partitions need a strategy. Newer
  non-blocking concurrency control targets streaming ingestion patterns.
- **Engine and version coupling is real.** Bundles are pinned to specific
  Spark/Flink and Scala versions (e.g. `hudi-spark3.5-bundle_2.12`); Spark 4.x
  needs Scala 2.13 and Java 17, and the Flink bundle cannot be built on Scala
  2.13. Upgrading the compute engine and upgrading Hudi are entangled.
- **The 0.x → 1.0 upgrade is a format change.** Hudi 1.0 (released late 2024)
  introduced timeline and log-format changes; migrating existing tables is a
  deliberate, tested operation, not a drop-in jar swap. Read the migration guide
  before upgrading production tables.

## When to Use / When Not

**Use when:**
- Your workload is upsert- or delete-heavy: CDC ingestion, GDPR deletes,
  dedup-on-write, dimension-table maintenance.
- You need incremental/CDC reads — pulling only what changed drives downstream
  pipelines far cheaper than full rescans.
- You want low-latency streaming ingestion into a lake (Flink or Spark
  structured streaming) with near-real-time query freshness via MoR.

**Avoid when:**
- Your tables are append-mostly analytics with rare mutations — Iceberg or plain
  Parquet is simpler and the query engine ecosystem is broader.
- Your team can't own background services and tuning; a mismanaged Hudi table
  degrades quietly.
- Your query engine of choice has thin Hudi support — read integration for
  Trino, Snowflake, BigQuery, etc. has historically lagged Iceberg's.

## Alternatives

- apache/iceberg — use instead when you want the widest query-engine and catalog
  support and your workload is analytics/append-heavy rather than upsert-heavy.
- delta-io/delta — use instead when you are Databricks/Spark-centric and want the
  most turnkey lakehouse table format with less operational tuning.
- apache/paimon — use instead when you are Flink-first and want a streaming
  lakehouse table format designed around a LSM/CDC model.
- apache/xtable — use alongside, not instead: it translates metadata between
  Hudi, Iceberg, and Delta so one physical table can be read as multiple formats.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016 | Created at Uber for incremental Hadoop ingestion[^1]. |
| — | 2017 | Open-sourced by Uber. |
| — | 2019-01 | Entered the Apache Incubator[^2]. |
| — | 2020-06 | Graduated to an Apache top-level project[^2]. |
| 0.9.0 | 2021-09 | Spark SQL DML, expanded metadata table. |
| 0.11.0 | 2022 | Multi-modal indexing, `spark-avro` no longer required at runtime. |
| 0.14.0 | 2023 | Record-level index, broader engine support. |
| 1.0.0 | 2024-12 | New timeline/log format; major on-disk change[^3]. |

## References

[^1]: Hudi documentation, "Overview" / project background. https://hudi.apache.org/docs/overview
[^2]: Apache Software Foundation, "The Apache Software Foundation Announces Apache Hudi as a Top-Level Project" — 2020-06-04. https://blogs.apache.org/foundation/entry/the-apache-software-foundation-announces64
[^3]: Apache Hudi releases page. https://hudi.apache.org/releases/release-1.0.0

## Tags

java, data-lakehouse, table-format, big-data, apache-spark, apache-flink, incremental-processing, stream-processing, cdc, datalake, upserts
