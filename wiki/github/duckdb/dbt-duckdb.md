# duckdb/dbt-duckdb

> The dbt adapter for DuckDB — run a full dbt project against an in-process
> analytics engine with no warehouse to provision.

[GitHub repo](https://github.com/duckdb/dbt-duckdb) ·
[License: Apache-2.0](https://github.com/duckdb/dbt-duckdb/blob/master/License.md)

## Overview

`dbt-duckdb` connects two projects: dbt, the SQL/Python transformation framework
that manages models, tests, and a build DAG, and DuckDB, an embedded columnar
OLAP engine that runs inside your process and reads CSV/JSON/Parquet files
directly. The adapter lets you point a dbt project at DuckDB the same way you
would point it at Snowflake or BigQuery — but with no server, no credentials to a
remote cluster, and no data-loading step. It was created by Josh Wills[^1] and
later moved under the DuckDB organization on GitHub (the old `jwills/dbt-duckdb`
path still redirects here).

The defining trade-off is that DuckDB is single-node and, on a persisted file,
single-writer. That makes `dbt-duckdb` excellent for local development, CI
pipelines, and the "modern data stack in a box"[^2] pattern where transforms run
over object storage without a warehouse — and a poor fit as a shared,
many-concurrent-user cloud warehouse. Much of the adapter's surface area
(external file sources, attachable databases, fsspec filesystems, MotherDuck)
exists to push past DuckDB's single-machine boundaries when you need to.

The adapter tracks `dbt-core` closely; the latest supported line targets
`dbt-core >= 1.8.x` with `duckdb` 1.1.x, and the maintainers state they work to
keep newer DuckDB releases working as they ship[^3].

## Getting Started

```bash
pip3 install dbt-duckdb
```

A minimal `profiles.yml` needs only a type; this runs against an ephemeral
in-memory database, which is the natural mode for CI or for pipelines that only
touch external files:

```yaml
default:
  outputs:
    dev:
      type: duckdb
      # path: /tmp/dbt.duckdb   # omit for an in-memory (:memory:) run
      threads: 4
  target: dev
```

```sql
-- models/orders.sql — read a Parquet file on S3 directly, no load step
select order_id, sum(amount) as total
from read_parquet('s3://my-bucket/orders/*.parquet')
group by 1
```

## Architecture / How It Works

`dbt-duckdb` is a standard dbt adapter: it implements dbt-core's Python adapter
interface (connection management, relation/catalog introspection, and a set of
Jinja macros for DDL/DML) and opens a DuckDB connection in the same process that
runs dbt. There is no network hop — models execute as SQL sent to an in-process
engine, and materializations (`table`, `view`, `incremental`, plus Python
models) map onto DuckDB `CREATE`/`INSERT` statements.

Because the engine is embedded, several capabilities are exposed straight
through the profile:

- **External files as first-class I/O.** Sources and models can reference
  CSV/JSON/Parquet via `external_location` (an f-string pattern per source), and
  models can be materialized as `external` to write Parquet/CSV back out. This
  is DuckDB's `read_parquet`/`COPY` surfaced through dbt.
- **Attached databases.** The `attach` profile key mounts additional databases
  into a run — other DuckDB files (including on S3), and via DuckDB's pluggable
  catalog engines also SQLite and Postgres, referenced by basename or alias[^4].
- **Extensions, settings, and secrets.** `extensions`, `settings`, and the
  DuckDB Secrets Manager (`secrets`, including the `credential_chain` provider
  and prefix-scoped credentials) are configured per profile and applied per
  connection.
- **fsspec filesystems.** An (experimental) `filesystems` block registers
  fsspec implementations (S3, GCS, Azure) with the connection.
- **A plugin system.** `dbt.adapters.duckdb.plugins` lets you register custom
  Python UDFs or source loaders; built-ins include `excel`, `gsheet`,
  `sqlalchemy`, `iceberg`, and an experimental `delta` plugin.

MotherDuck — a managed, cloud DuckDB — is reached by setting `path` to an
`md:` connection string, which turns the otherwise-local engine into a hosted,
multi-user one. Hosted DuckLake on MotherDuck is supported via `is_ducklake`.

## Production Notes

**Single-writer file lock is the central footgun.** A persisted `.duckdb` file
can be opened for write by only one process at a time. Two concurrent dbt
invocations against the same file — a scheduler firing overlapping runs, or a
BI tool holding the file open — will collide. The adapter's `retries` block
(`connect_attempts`, `query_attempts`, `retryable_exceptions`) exists partly to
wait out a held lock, but it is a mitigation, not concurrency. The real answers
are: run in-memory over external files, give each job its own file, or use
MotherDuck for shared write access.

**It is memory- and machine-bound.** DuckDB is single-node; a large join or
aggregation can exhaust RAM. Tune `memory_limit`/`threads` in `settings`, and
remember there is no horizontal scaling — a model that outgrows the box has to
be restructured, not sharded across a cluster.

**Storage-format and version coupling.** DuckDB's on-disk format stabilized at
1.0; across engine upgrades a `.duckdb` file may need to be rebuilt, so pin the
`duckdb` version in CI and treat persisted files as reproducible artifacts, not
long-lived state. The adapter's supported range also follows `dbt-core` minors,
so upgrades are usually done in lockstep with dbt.

**MotherDuck lags the local client.** MotherDuck is compatible with specific
(older) client DuckDB versions and does not support loading custom extensions or
user-defined functions, and preloads only common extensions — so a pipeline that
works locally with a bleeding-edge DuckDB or a custom UDF may not run there.

**Plugins pull real dependencies.** `excel` needs pandas + openpyxl/xlsxwriter,
`gsheet` needs gspread, `iceberg` needs pyiceberg and Python ≥ 3.10, and Python
models execute in the dbt process — so their imports become your environment's
dependencies. Experimental features (`delta`, fsspec filesystems) can change
between releases.

## When to Use / When Not

**Use when:**
- You want dbt's modeling/testing/docs layer without provisioning a warehouse.
- Your pipeline reads or writes Parquet/CSV/JSON on object storage — a lakehouse
  without a compute cluster.
- You need fast, cheap dbt runs in CI, or a local dev target that mirrors prod.
- Data fits on one machine and writes are effectively single-process.

**Avoid when:**
- Many users or jobs must write the same database concurrently (use a cloud
  warehouse or MotherDuck).
- Data or working sets exceed a single node's memory and can't be partitioned.
- You need a persistent, always-on, multi-tenant SQL service rather than a
  batch transform engine.

## Alternatives

- dbt-labs/dbt-core — with dbt-snowflake / dbt-bigquery / dbt-redshift when you
  need a shared, concurrent, horizontally scaled cloud warehouse.
- duckdb/duckdb — use the engine directly when you don't need dbt's DAG,
  testing, or documentation layer.
- TobikoData/sqlmesh — an alternative transformation framework (native DuckDB
  support, virtual data environments, column-level lineage) when you want
  stronger state management than dbt's.
- ibis-project/ibis — a Python dataframe API over DuckDB and other backends when
  you prefer composing Python to SQL + Jinja.
- MotherDuck — managed, serverless DuckDB when you specifically need the hosted,
  multi-user concurrency that a local file can't provide.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-09 | Repo created by Josh Wills[^1]. |
| 1.4.x | 2023 | `attach` for additional databases; experimental fsspec `filesystems`. |
| 1.5.2 | 2023 | MotherDuck support via `md:` connection strings. |
| 1.6.0 | 2023 | `module_paths` for local Python modules. |
| 1.8.x | 2024+ | Current supported line; targets `dbt-core >= 1.8.x`, `duckdb` 1.1.x[^3]. |
| 1.9.6 | 2025 | Hosted DuckLake on MotherDuck via `is_ducklake`. |

Version numbers track dbt-core's minor cadence; feature-to-version mappings above
are from the project README, exact release days are approximate.

## References

[^1]: Adapter authored by Josh Wills; repository now hosted under the DuckDB org
(the former `jwills/dbt-duckdb` path redirects). https://github.com/duckdb/dbt-duckdb
[^2]: "Modern Data Stack in a Box with DuckDB", DuckDB blog, 2022-10-12.
https://duckdb.org/2022/10/12/modern-data-stack-in-a-box.html
[^3]: dbt-duckdb README — supported `dbt-core` / `duckdb` versions.
https://github.com/duckdb/dbt-duckdb#dbt-duckdb
[^4]: DuckDB `ATTACH` documentation.
https://duckdb.org/docs/sql/statements/attach.html

## Tags

python, dbt, duckdb, data-engineering, analytics-engineering, olap, elt, data-transformation, lakehouse, parquet, adapter, motherduck
