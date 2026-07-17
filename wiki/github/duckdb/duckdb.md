# duckdb/duckdb

> An in-process analytical (OLAP) SQL database — "SQLite for analytics": columnar, vectorized, embedded, zero external dependencies.

[GitHub repo](https://github.com/duckdb/duckdb) ·
[Official website](https://duckdb.org) ·
[License: MIT](https://github.com/duckdb/duckdb/blob/main/LICENSE)

## Overview

DuckDB is an embedded analytical database that runs inside your host process rather than as a separate server. It was started in 2018 by Mark Raasveldt and Hannes Mühleisen at CWI (Centrum Wiskunde & Informatica) in Amsterdam, and is developed today by DuckDB Labs with IP held by the non-profit DuckDB Foundation[^1]. The design goal is the analytical mirror image of SQLite: no server to run, no connection to manage, a single-file (or in-memory) database, but with a column-store and a vectorized execution engine tuned for scans, joins, and aggregations over large tables.

The defining tension is **embedded OLAP vs. concurrency**. DuckDB is extremely fast for a single analyst or a single application process crunching millions to billions of rows on one machine, and it reads Parquet/CSV/JSON and zero-copies pandas/Arrow/Polars data frames without an import step. But it is not a multi-user server: a database file allows one read-write process at a time. It is a query engine and storage format, not a shared backend.

DuckDB is most used for: local data analysis (a faster, SQL-native alternative to pandas), ETL/ELT transformation steps, embedded analytics inside applications, querying data lakes (Parquet on S3 via the `httpfs` extension), and as the engine under other tools. A separate company, MotherDuck (founded by ex-BigQuery lead Jordan Tigani), sells a hosted/hybrid cloud built on DuckDB[^2] — it is a distinct commercial entity, not part of this repository.

## Getting Started

```bash
# CLI (macOS/Linux)
curl https://install.duckdb.org | sh
# Python
pip install duckdb
```

```python
import duckdb

# Query a Parquet file directly — no load step, no schema declaration
duckdb.sql("SELECT country, SUM(revenue) FROM 'sales/*.parquet' GROUP BY country")

# Zero-copy over a pandas DataFrame by variable name
import pandas as pd
df = pd.read_csv("orders.csv")
duckdb.sql("SELECT status, COUNT(*) FROM df GROUP BY status").show()

# Persistent database file
con = duckdb.connect("analytics.duckdb")
con.execute("CREATE TABLE t AS SELECT * FROM read_parquet('events.parquet')")
```

## Architecture / How It Works

DuckDB is a single C++17 codebase with no required third-party dependencies in its core, which is why it embeds cleanly into Python wheels, R packages, a WebAssembly build, and mobile apps. The engine has four load-bearing design choices:

1. **Columnar storage.** Data is stored column-by-column, both in its native single-file format and when scanning Parquet. Analytical queries touch few columns of many rows, so column layout minimizes I/O and enables per-column compression.
2. **Vectorized execution.** Instead of processing one row at a time (classic Volcano) or compiling queries to machine code, DuckDB processes batches ("vectors", ~2048 values) through operators[^3]. This amortizes interpretation overhead while keeping data in CPU cache — a middle path between tuple-at-a-time and full JIT.
3. **Morsel-driven parallelism.** Scans are split into "morsels" distributed across threads, with a push-based pipeline model, giving intra-query multi-core parallelism on a single node without a distributed scheduler.
4. **MVCC + single-file storage.** Writes use multi-version concurrency control; the on-disk format bundles the whole database (tables, indexes, metadata) into one file with ACID transactions.

External data is a first-class citizen: table functions like `read_parquet`, `read_csv`, and `read_json` let SQL run over files without loading them into DuckDB's own format. The extension system (`INSTALL`/`LOAD`) adds `httpfs` (S3/HTTP), `spatial`, `json`, `iceberg`, `postgres`/`mysql`/`sqlite` scanners, and full-text search as loadable modules versioned against the core.

## Production Notes

**Concurrency is the number-one footgun.** A DuckDB database file supports one read-write process at a time. Multiple threads inside that one process are fine and parallelized automatically, but you cannot point two application servers at the same `.duckdb` file for concurrent writes. Read-only multi-process access is possible; a read-write server pattern is not what DuckDB is for. Teams that treat it as a shared backend hit lock errors immediately.

**Larger-than-memory, but not unconditionally.** DuckDB can spill intermediate results to disk, so joins and aggregations can exceed RAM. But some operations remain memory-hungry, and default behavior on a laptop can still OOM on large sorts/joins. Set `SET memory_limit` and `SET threads`, and provide a real `temp_directory` for spilling. Performance is best when the working set fits in memory.

**Storage format stability.** Before 1.0.0 (June 2024), the on-disk format could change between minor releases, sometimes requiring export/re-import to upgrade. Since 1.0.0, backward compatibility of the storage format is a stated commitment[^4] — but pre-1.0 database files and mismatched extension versions are still a real upgrade hazard. Pin your DuckDB version in production and test format upgrades.

**Extensions are version-locked.** Loadable extensions are built against a specific DuckDB version; upgrading the core can require re-fetching extensions, and autoloading pulls from a network repository unless you vendor them. Air-gapped deployments need to pre-stage extension binaries.

**Not an OLTP database.** High-frequency single-row inserts/updates are slow relative to a row-store; the columnar format and MVCC are optimized for bulk analytical workloads. Use it as a read/analytics engine, not a transactional system of record.

## When to Use / When Not

**Use when:**
- You want SQL over Parquet/CSV/JSON/Arrow/pandas on one machine, fast, without standing up a server.
- You're building an ETL/transform step, notebook analysis, or embedded in-app analytics.
- You need to query data-lake files (S3 Parquet) interactively without a warehouse.
- You want a portable single-file analytical store, or a browser/WASM analytics engine.

**Avoid when:**
- You need many processes writing concurrently, or a shared multi-user server — use a client-server DBMS.
- Your workload is OLTP: frequent small transactional writes and point updates.
- Your data genuinely doesn't fit on one large node and needs a distributed cluster.
- You require managed cloud scaling, replication, and HA out of the box (consider MotherDuck or a cloud warehouse).

## Alternatives

- sqlite/sqlite — the row-store embedded database DuckDB is patterned after; use it for OLTP and transactional app storage, not analytics.
- ClickHouse/ClickHouse — columnar OLAP as a scalable server/cluster; use it when you need a distributed, multi-user analytical backend.
- pola-rs/polars — in-process DataFrame engine (Rust/Arrow); use it when you want a DataFrame API rather than SQL and tight Python/Rust integration.
- apache/arrow (DataFusion) — Rust query engine and columnar standard; use it to embed a customizable SQL/query engine in Rust services.
- apache/druid — real-time analytics server for streaming ingestion and low-latency dashboards at scale; use it for always-on multi-tenant serving.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2019 | First public release out of CWI[^1]. |
| 0.9.0 | 2023-09 | Storage improvements; larger-than-memory processing advances. |
| 1.0.0 "Snow Duck" | 2024-06-03 | First stable release; on-disk format backward-compatibility commitment[^4]. |
| 1.1.0 | 2024-09 | Post-1.0 feature release. |
| 1.2.0 | 2025-02 | Performance and SQL feature additions. |
| 1.3.0 "Ossivalis" | 2025-05 | Continued 1.x line. |

## References

[^1]: DuckDB, "About DuckDB / Foundation." https://duckdb.org/foundation/
[^2]: MotherDuck — hosted DuckDB service, separate company. https://motherduck.com/
[^3]: Raasveldt & Mühleisen, "DuckDB: an Embeddable Analytical Database" (SIGMOD 2019 demo). https://duckdb.org/pdf/SIGMOD2019-demo-duckdb.pdf
[^4]: DuckDB blog, "Announcing DuckDB 1.0.0" — 2024-06-03. https://duckdb.org/2024/06/03/announcing-duckdb-100.html

## Tags

c-plus-plus, database, olap, analytics, embedded-database, sql, columnar, vectorized, in-process, data-analysis
