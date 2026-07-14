# apache/iceberg

> An open table format that gives huge analytic datasets on object storage the ACID semantics, schema evolution, and time travel of a SQL table — without locking them to one engine.

[GitHub repo](https://github.com/apache/iceberg) ·
[Official website](https://iceberg.apache.org) ·
[License: Apache-2.0](https://github.com/apache/iceberg/blob/main/LICENSE)

## Overview

Iceberg is a *table format*, not a storage engine or a query engine. It is a specification for how to lay out metadata on top of data files (Parquet, ORC, Avro) sitting in object storage or HDFS, so that many engines — Spark, Trino, Flink, Presto, Hive, Impala, Dremio, Snowflake, BigQuery — can read and write the same tables concurrently with correct isolation[^1]. The project originated at Netflix (Ryan Blue, Dan Weeks) to fix the correctness and scale problems of Hive tables, was donated to the Apache Software Foundation, and became a top-level project in 2020[^2].

The core problem Iceberg solves is that "a table = a directory of files" (the Hive model) has no atomic commit, relies on expensive directory listings, and makes schema/partition changes a rewrite. Iceberg replaces directory-listing with an explicit, versioned metadata tree, so a commit is a single atomic pointer swap and a scan is a metadata read instead of a `LIST` over millions of objects.

The defining tension is that Iceberg is deliberately *just a spec plus a reference Java library*. That neutrality is its whole value proposition — no single vendor owns your tables — but it also means correctness and feature parity depend on each engine's implementation. The same table can behave differently, or refuse newer spec features, depending on which engine and catalog you point at it. This repository is the Java reference implementation; Python (PyIceberg), Rust, Go, and C++ live in sibling repos[^3] and lag the Java feature set to varying degrees.

## Getting Started

Iceberg is consumed through an engine. The most common entry point is Spark via the runtime jar:

```bash
spark-sql \
  --packages org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.9.0 \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
  --conf spark.sql.catalog.local=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.local.type=hadoop \
  --conf spark.sql.catalog.local.warehouse=$PWD/warehouse
```

```sql
CREATE TABLE local.db.events (
  id     BIGINT,
  ts     TIMESTAMP,
  level  STRING
) USING iceberg
PARTITIONED BY (days(ts));          -- hidden partition transform

INSERT INTO local.db.events VALUES (1, TIMESTAMP '2026-07-14 10:00:00', 'info');

SELECT * FROM local.db.events FOR SYSTEM_VERSION AS OF 1;   -- time travel by snapshot
```

Python users skip the JVM entirely with PyIceberg (`pip install "pyiceberg[pyarrow]"`), which reads/writes tables and talks to catalogs directly.

## Architecture / How It Works

Iceberg's metadata is a tree, read top-down at query planning time:

1. **Catalog** — maps a table name to the location of its current *metadata file*. The atomic commit is a compare-and-swap on this pointer. Catalog implementations: REST (the recommended, engine-neutral protocol), Hive Metastore, JDBC, AWS Glue, Nessie, Snowflake, and the file-based Hadoop catalog (discouraged for concurrent writes).
2. **Table metadata file** (JSON) — schema(s), partition spec(s), sort orders, snapshot list, and current snapshot. Every commit writes a *new* metadata file.
3. **Manifest list** (Avro) — one per snapshot; points to manifest files with per-manifest partition ranges for pruning.
4. **Manifest files** (Avro) — list data files (and delete files) with column-level stats (min/max/null counts) used to skip files at plan time.
5. **Data files** — Parquet/ORC/Avro on object storage.

Key design choices that follow from this structure:

- **Hidden partitioning.** Partition values are derived from source columns via transforms (`days(ts)`, `bucket(16, id)`, `truncate`). Queries filter on the raw column and Iceberg computes partition pruning itself — no `WHERE dt='...'` boilerplate, and the partition scheme can evolve without rewriting old data[^1].
- **Schema evolution by column ID.** Columns are tracked by a stable integer ID, not by name or position, so rename/reorder/add/drop are pure metadata operations.
- **Snapshots** give time travel and rollback for free — every write is a new snapshot; old ones remain until expired.
- **Row-level deletes.** Format spec v2 introduced *merge-on-read* via position and equality delete files; v3 replaces position deletes with binary *deletion vectors* stored in Puffin files, plus row lineage and new types (variant, geometry/geography)[^4]. Each writer chooses copy-on-write (rewrite whole data files on update) or merge-on-read (write delete files, reconcile at read time).

The spec is the source of truth; the Java library is the reference. Cross-engine correctness hinges on every implementation honoring the same metadata semantics — which is why the *format version* of a table gates which engines can touch it.

## Production Notes

**Table maintenance is not optional.** Streaming or frequent writes produce many small data files and manifest bloat, degrading scan planning. You must schedule `rewrite_data_files` (compaction), `rewrite_manifests`, `expire_snapshots`, and `remove_orphan_files`. Neglecting `expire_snapshots` means old data files are never garbage-collected and storage grows unbounded; running it too aggressively breaks in-flight time-travel reads and any downstream incremental consumers.

**Catalog choice is a lock-in decision, not a config detail.** The Hadoop (file-based) catalog has no safe atomic commit on S3 without an external lock and is fine only for single-writer or demo use. Production deployments should use a REST catalog or a metastore-backed one. Migrating catalogs later means re-registering every table.

**Commit conflicts under concurrency.** Commits are optimistic — a writer reads the current snapshot, stages files, and CAS-swaps the pointer. Under many concurrent writers to the same table, commits retry and can starve; partition your write workloads or serialize them.

**Merge-on-read vs copy-on-write is a real tradeoff.** MoR makes writes cheap and reads expensive (readers merge delete files); CoW does the opposite. Choosing wrong for your read/write ratio is a common performance surprise, and delete-file accumulation under MoR requires regular compaction to stay readable.

**Spec-version and engine skew.** A table upgraded to format v3 (deletion vectors, new types) can become unreadable to an engine that only speaks v2. Because engines ship Iceberg support on their own cadence, "which features can I use" is really "which is supported by the *oldest* engine in my stack." Verify per-engine before upgrading a table's format version.

**PyIceberg is not at Java parity.** It has advanced quickly for reads and basic writes, but historically lagged on write features, compaction, and some catalog types. Confirm the specific operation you need is supported rather than assuming Java-equivalence.

## When to Use / When Not

**Use when:**
- Multiple engines (e.g. Spark for ETL, Trino for BI, Flink for streaming) must read/write the *same* tables safely.
- You need ACID commits, schema/partition evolution, time travel, or rollback on data-lake-scale datasets.
- You want to avoid warehouse lock-in and keep data as open files on object storage (the "lakehouse" pattern).
- Your Hive tables have outgrown directory-listing performance or suffer from non-atomic writes.

**Avoid when:**
- Your dataset is small enough for a normal database or a single Parquet file — the metadata machinery is overhead.
- You have one engine and one writer and no need for evolution/time-travel — a plain warehouse is simpler.
- You cannot commit to running maintenance jobs; an unmaintained Iceberg table degrades and leaks storage.
- You need sub-second OLTP-style updates — Iceberg targets analytic batch/stream workloads, not transactional point mutations.

## Alternatives

- delta-io/delta — Delta Lake; comparable table format, historically tightest with Spark/Databricks. Use instead when your stack is Databricks-centric; the two formats are converging and increasingly interoperable.
- apache/hudi — optimized for streaming ingestion, upserts, and incremental pulls with built-in indexes. Use when heavy record-level upserts and CDC are the primary workload.
- apache/paimon — Flink-native streaming lakehouse format. Use when Flink streaming ingestion with fast updates is the center of gravity.
- apache/polaris — not an alternative format but an open Iceberg REST catalog. Use alongside Iceberg when you need a vendor-neutral catalog service.
- Plain Hive tables — the thing Iceberg replaces. Only "use instead" if you cannot change legacy tooling.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-sourced | 2018-11 | Released by Netflix; incubating at Apache[^2]. |
| Top-level project | 2020-05 | Graduated to Apache TLP[^2]. |
| Format spec v2 | 2021 | Row-level deletes via merge-on-read (position + equality delete files). |
| 1.0.0 | 2022-11 | First 1.0 Java release; API stability commitment[^5]. |
| REST catalog | 2022–2023 | Open catalog protocol standardized for engine-neutral commits. |
| Tabular acquired | 2024-06 | Databricks acquired Tabular (founded by Iceberg's creators); accelerated Delta/Iceberg convergence[^6]. |
| Format spec v3 | 2025 | Deletion vectors (Puffin), row lineage, variant and geo types[^4]. |

## References

[^1]: Apache Iceberg documentation and table spec. https://iceberg.apache.org/spec/
[^2]: Apache Iceberg project background and status. https://iceberg.apache.org/
[^3]: Non-Java implementations: apache/iceberg-python, apache/iceberg-rust, apache/iceberg-go, apache/iceberg-cpp. https://github.com/apache/iceberg-python
[^4]: Iceberg table format specification (v2/v3 features, deletion vectors, types). https://iceberg.apache.org/spec/
[^5]: Iceberg releases. https://iceberg.apache.org/releases/
[^6]: Databricks, "Databricks + Tabular" — 2024-06. https://www.databricks.com/blog/databricks-tabular

## Tags

java, table-format, lakehouse, data-lake, big-data, apache, acid, spark, trino, flink, analytics, object-storage
