# delta-io/delta

> An open table format that puts ACID transactions and time travel on top of Parquet files in object storage — the reference implementation of the Delta Lake protocol.

[GitHub repo](https://github.com/delta-io/delta) ·
[Official website](https://delta.io) ·
[License: Apache-2.0](https://github.com/delta-io/delta/blob/master/LICENSE.txt)

## Overview

Delta Lake is a storage framework that turns a directory of Parquet files in object storage (S3, ADLS, GCS, HDFS) into a table with ACID transactions, schema enforcement, upserts, and time travel. It was built at Databricks and open-sourced in 2019[^1]; the project is now hosted under the Linux Foundation[^2]. This repository (`delta-io/delta`) is the JVM reference implementation — the Spark connector, the Delta Standalone Java library, the Flink/Hive connectors, and the canonical protocol spec all live here.

The mechanism is a write-ahead log. Alongside the Parquet data files, Delta keeps a `_delta_log/` directory of ordered JSON commit files; each commit lists the files added and removed by a transaction. Readers reconstruct table state by replaying the log (accelerated by periodic Parquet checkpoints), and writers commit with optimistic concurrency control, retrying on conflict. This gives serializable isolation without a database server — the correctness guarantees are borrowed from the atomicity and mutual-exclusion properties of the underlying storage system[^3].

The defining tension is Spark coupling. For most of its life "Delta Lake" effectively meant "Delta on Spark," and the richest feature set (MERGE, OPTIMIZE, Z-ordering, deletion vectors) is still delivered through the Spark connector. The project has spent recent releases decoupling — Delta Kernel (a narrow engine-agnostic library) and UniForm (exposing Delta tables as Iceberg/Hudi metadata) are both attempts to be an open format rather than a Spark feature — but engine parity remains uneven, and the separate `delta-io/delta-rs` project exists precisely because non-JVM engines needed a path that did not route through this codebase.

## Getting Started

```bash
pip install delta-spark          # Python + bundled Spark connector
```

```python
from pyspark.sql import SparkSession
from delta import configure_spark_with_delta_pip

builder = (
    SparkSession.builder.appName("delta-quickstart")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog",
            "org.apache.spark.sql.delta.catalog.DeltaCatalog")
)
spark = configure_spark_with_delta_pip(builder).getOrCreate()

# Write a Delta table
spark.range(0, 5).write.format("delta").save("/tmp/delta-table")

# Upsert (MERGE) and read a prior version (time travel)
from delta.tables import DeltaTable
DeltaTable.forPath(spark, "/tmp/delta-table")  # exposes merge/update/delete/vacuum

spark.read.format("delta").option("versionAsOf", 0).load("/tmp/delta-table").show()
```

The JVM build uses SBT and requires JDK 17 to compile (`build/sbt compile`); the Python test path pins JDK 11[^4].

## Architecture / How It Works

A Delta table is a storage path containing:

1. **Data files** — Parquet, optionally partitioned into subdirectories. Delta never mutates a data file in place; updates rewrite files and record the swap in the log.
2. **The transaction log** (`_delta_log/`) — monotonically numbered JSON files (`00000000000000000000.json`, `...001.json`, …). Each is a set of *actions*: `add`/`remove` file, `metaData` (schema, partitioning), `protocol` (reader/writer requirements), `commitInfo`, and `txn` (idempotent streaming markers).
3. **Checkpoints** — every N commits (default 10) Delta writes a Parquet summary of the full table state so readers do not replay the entire log from zero.

**Concurrency** is optimistic. A writer reads the current log version, stages its changes, and attempts to write commit `n+1`. If another writer already claimed `n+1`, it re-reads, checks whether the conflict is logically compatible, and retries. Serializability depends on exactly one writer being able to create a given commit file — the LogStore abstraction. On HDFS/Azure this is a native atomic rename; on S3, which historically lacked put-if-absent, multi-cluster writes required an external coordinator (the DynamoDB-backed `S3DynamoDBLogStore`)[^3]. S3 conditional writes have since made single-coordinator setups less necessary, but the footgun is old and widely deployed.

**Protocol table features.** Early Delta used integer minimum reader/writer versions. Delta 3.0 replaced the coarse version numbers with *named table features* (e.g. `deletionVectors`, `columnMapping`, `timestampNtz`), so an engine can advertise exactly which capabilities it supports and refuse tables that need one it lacks[^5]. **Deletion vectors** implement merge-on-read: a DELETE/UPDATE marks rows as removed in a side file rather than rewriting the whole Parquet file, trading read-time work for faster writes. **Delta Kernel** is a smaller Java/Rust library that lets other engines read/write Delta without pulling in Spark. **UniForm** writes Iceberg (and Hudi) metadata alongside the Delta log so Iceberg readers can query the same Parquet files[^6].

## Production Notes

**The small-files problem is the operational core of running Delta.** Streaming and frequent micro-batch writes produce many tiny Parquet files; read performance and log replay both degrade. You must run `OPTIMIZE` (bin-packing compaction), optionally with `ZORDER BY` for multi-dimensional locality, on a schedule. Without it, a busy table slowly gets slower.

**VACUUM deletes tombstoned files and breaks time travel past its retention window** (default 7 days). Running `VACUUM` with a short retention permanently removes the files older versions point to; queries with `versionAsOf`/`timestampAsOf` beyond the horizon then fail. The safety check that blocks sub-7-day retention exists because someone always tries to disable it.

**Metadata scales with file count, not data size.** Tables with millions of files have large logs and large checkpoints; state reconstruction and planning become the bottleneck. Mitigations: frequent checkpointing, log/checkpoint compaction, partitioning that does not explode into tiny partitions, and OPTIMIZE to keep file counts down.

**Concurrent writers conflict.** Two overlapping MERGE/UPDATE transactions on the same files will make one of them fail its optimistic-commit retry. Delta is serializable, not lock-free — high-contention write patterns need partitioning that isolates writers, or a single-writer design.

**Engine parity is a real constraint.** A table written by the Spark connector using deletion vectors, column mapping, or the newest table features may be unreadable by an older Trino, Presto, Flink, or `delta-rs` build until that engine implements the feature. Check the reader/writer feature set before enabling protocol upgrades on a table other systems consume — protocol upgrades are one-way.

**Version and JDK churn.** Each Delta release targets a specific Apache Spark line (e.g. Delta 2.x → Spark 3.x, Delta 4.x → Spark 4.x); you do not get to mix freely. Confirm the Delta/Spark compatibility matrix before upgrading either[^7].

## When to Use / When Not

**Use when:**
- You run Spark (or Databricks) and need ACID upserts, deletes, and MERGE on a data lake.
- You need time travel, auditable history, or reproducible reads of a prior table version.
- You want schema enforcement and evolution on Parquet without moving to a warehouse.
- You have streaming ingestion that must be transactionally consistent with batch reads.

**Avoid when:**
- Your stack is non-JVM and Iceberg has better native support in your engines (Trino, Flink, Snowflake, DuckDB) — evaluate `apache/iceberg` first.
- You want a lightweight Rust/Python path with no Spark — `delta-io/delta-rs` fits better.
- Your workload is low-latency point lookups or high-concurrency OLTP — Delta is an analytical table format, not an operational database.
- You have few, large, append-only files and no need for updates — plain Parquet may be enough.

## Alternatives

- apache/iceberg — competing open table format with broader multi-engine native support and hidden partitioning; use instead when your query engines (Trino, Flink, Snowflake) target Iceberg first.
- apache/hudi — table format oriented around streaming upserts and incremental pulls; use when record-level upsert throughput and near-real-time ingestion dominate.
- delta-io/delta-rs — Rust implementation of the Delta protocol with Python bindings; use when you want Delta without the JVM/Spark stack (pandas, Polars, DataFusion).
- apache/parquet-format — the underlying columnar file format; use plain Parquet when you need neither transactions nor mutation.
- Apache Paimon — streaming-first lakehouse format from the Flink ecosystem; use when Flink CDC ingestion is the primary workload.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2019-04 | Initial open-source release by Databricks[^1]. |
| 0.7.0 | 2020-06 | Spark 3.0 support, SQL DDL/DML via catalog. |
| 1.0.0 | 2021-05 | First 1.x; generated columns, `MERGE` improvements. |
| 2.0.0 | 2022-08 | Open-sourced previously Databricks-only features (Z-order, data skipping)[^8]. |
| 2.3.0 | 2023-04 | Deletion vectors (read path), shallow clone. |
| 3.0.0 | 2023-10 | Table features, Delta Kernel, UniForm, Liquid Clustering preview[^5][^6]. |
| 4.0.0 | 2025 | Spark 4.0 line, Delta Connect, variant type, type widening, coordinated commits[^7]. |

## References

[^1]: Databricks, "Open Sourcing Delta Lake" — 2019-04-24. https://databricks.com/blog/2019/04/24/open-sourcing-delta-lake.html
[^2]: Linux Foundation / Delta Lake project home. https://delta.io/
[^3]: Delta Lake docs, "Storage configuration" (LogStore, S3 multi-cluster, storage requirements). https://docs.delta.io/latest/delta-storage.html
[^4]: Repository README — Building (SBT, JDK 17) and Python tests (JDK 11). https://github.com/delta-io/delta
[^5]: Delta Transaction Log Protocol — table features and reader/writer versions. https://github.com/delta-io/delta/blob/master/PROTOCOL.md
[^6]: Delta Lake, "Delta Lake 3.0" (Universal Format / UniForm, Delta Kernel). https://delta.io/blog/delta-lake-3-0/
[^7]: Delta Lake releases and Spark compatibility. https://docs.delta.io/latest/releases.html
[^8]: Delta Lake, "Delta Lake 2.0" announcement. https://delta.io/blog/2022-08-02-delta-2-0-the-foundation-of-your-data-lake-is-open/

## Tags

scala, spark, lakehouse, table-format, acid, data-engineering, big-data, parquet, transaction-log, time-travel, delta-lake, storage
