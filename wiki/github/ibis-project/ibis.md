# ibis-project/ibis

> A portable Python dataframe API that compiles the same expressions to 20+ SQL and dataframe backends.

[GitHub repo](https://github.com/ibis-project/ibis) ·
[Official website](https://ibis-project.org) ·
[License: Apache-2.0](https://github.com/ibis-project/ibis/blob/main/LICENSE.txt)

## Overview

Ibis is a Python library that gives you a single, pandas-adjacent dataframe API and defers execution to whatever backend you point it at — DuckDB locally, BigQuery/Snowflake/Trino/ClickHouse in production, Polars or DataFusion for in-process columnar work[^1]. You write lazy expressions in Python; Ibis compiles them to the backend's native language (SQL for most, a dataframe plan for a few) and only runs when you ask for results. It was created by Wes McKinney (the author of pandas) at Cloudera in 2015 as a Python front end for Impala and SQL-on-Hadoop, and later saw sustained development funded by Voltron Data.

The defining promise is portability: "iterate locally, deploy remotely by changing one line." The defining tension is that portability is *aspirational, not total*. A given expression may compile and run on DuckDB and then raise `OperationNotDefinedError` on a backend that lacks the underlying operation, and SQL-dialect differences (NULL ordering, string/timestamp semantics, type coercions) leak through the abstraction. Ibis narrows the gap between backends far more than writing raw SQL by hand would, but it does not erase it.

The other thing to know going in is API churn. Ibis has shipped many breaking major releases, and upgrading across them (3 → 4 → … → 8 → 10) has repeatedly required code changes. The 8.0 release (2024) replaced the long-standing SQLAlchemy-based compiler with a SQLGlot-based one — a large internal rewrite that expanded backend coverage but changed generated SQL and behavior[^3][^4]. Treat Ibis as a fast-moving library, not a frozen standard.

## Getting Started

```bash
pip install 'ibis-framework[duckdb,examples]'
```

```python
import ibis

ibis.options.interactive = True          # eager, pretty repr in a REPL/notebook

t = ibis.examples.penguins.fetch()        # DuckDB-backed example table
g = (
    t.group_by("species", "island")
     .agg(count=t.count())
     .order_by("count")
)
g                                         # triggers a LIMITed query and prints

print(ibis.to_sql(g))                     # inspect the compiled SQL without running
```

Switch execution engines without rewriting the expression:

```python
con = ibis.duckdb.connect()               # or ibis.polars.connect(), ibis.bigquery.connect(...)
t = con.read_parquet("penguins.parquet")
con.execute(t.group_by("species").agg(n=t.count()))
```

## Architecture / How It Works

Ibis is a three-layer system: an **expression API**, an **operation graph (IR)**, and per-backend **compilers**.

1. **Expressions** (`ibis.expr.types`) — what you touch. `Table`, `Column`, `Scalar`. They are immutable and lazy; every method returns a new expression rather than mutating data. Nothing executes until you call `.execute()` / `.to_pandas()` / `.to_pyarrow()` or trigger interactive repr.
2. **Operations** (`ibis.expr.operations`) — each expression wraps a node in a typed, immutable operation graph. Nodes carry a resolved output type from Ibis's own datatype system (`dt.int64`, `dt.Struct`, `dt.Array`, …), which is validated at construction time. Building an expression that a backend can't type will fail in Python, before any query runs.
3. **Compilers** — a backend turns the op graph into something executable. Ibis splits backends into two families: **SQL-generating** (BigQuery, Snowflake, Postgres, DuckDB, Trino, ClickHouse, SQLite, MySQL, MSSQL, Spark, Flink, and more) and **dataframe/DataFusion-generating** (Polars, DataFusion). SQL backends share a single compiler that emits a SQLGlot AST and prints it in the target dialect[^4]; this is what the 8.0 rewrite unified. Dataframe backends translate the op graph to that engine's native plan instead.

Two consequences fall out of this design. First, the generated SQL is deliberately correct-over-pretty: Ibis emits heavily nested subqueries/CTEs with generated aliases (`t0`, `t1`, …), leaning on the backend's optimizer to flatten them. It runs fine but is verbose to read when debugging. Second, an operation only works on a backend if that backend's compiler implements it — the surface area is a matrix of (operation × backend), not a single guaranteed set.

The escape hatch is `.sql()` / `Table.sql()`, which lets you drop raw backend SQL into the middle of an Ibis pipeline and keep composing. This is the intended pressure valve for anything the abstraction doesn't cover.

## Production Notes

- **Portability is per-operation, not per-backend.** Code that runs on DuckDB can raise `OperationNotDefinedError` on another backend. If you plan to swap engines, test the actual expressions against each target — don't assume "Ibis supports X" means "X supports my query." The per-backend operation-support pages are the source of truth[^2].
- **Interactive mode executes on every repr.** `ibis.options.interactive = True` makes each display of an expression run a `LIMIT`ed query against the backend. Convenient in notebooks; on a metered warehouse (BigQuery/Snowflake) every stray repr is a billable query. Keep it off in scripts and jobs.
- **Dialect semantics still leak.** NULL sort order, integer vs. float division, string collation, and timestamp/timezone handling follow the *backend's* rules, not a normalized Ibis rule. Same expression, subtly different results across engines.
- **The pandas and Dask backends were removed.** They were deprecated in the 9.x line and dropped afterward; local/in-memory execution now goes through DuckDB by default[^3]. Code that depended on the pandas backend's eager, row-by-row Python semantics does not port directly — DuckDB executes as SQL.
- **Upgrades are real work.** Cross-major upgrades have repeatedly changed APIs and, in 8.0, the generated SQL itself. Pin `ibis-framework` and read the release notes before bumping; budget time for a migration on majors.
- **Compilation is cheap; execution is not.** Ibis's own compile overhead is negligible next to query runtime. Performance is almost entirely the backend's — Ibis neither speeds up nor slows down DuckDB/BigQuery meaningfully. Use `ibis.to_sql()` to inspect what will actually run.
- **Install extras per backend.** The base package is thin; each backend is an optional extra (`ibis-framework[duckdb]`, `[bigquery]`, `[snowflake]`, …) pulling that engine's driver. Mismatched or missing extras are the most common first-run error.

## When to Use / When Not

**Use when:**
- You target more than one execution engine, or want to develop against local DuckDB and deploy the same code to a cloud warehouse.
- You want dataframe ergonomics but need the work to push down into a database rather than pull data into Python memory.
- You want to compose Python and raw SQL in one pipeline, or generate SQL programmatically and inspect it.
- Your team knows pandas-style APIs and your data lives in a SQL engine.

**Avoid when:**
- You target exactly one backend, are happy with its native SDK/SQL, and don't value the abstraction — you're paying churn for portability you won't use.
- You need every exotic backend-specific function first-class (you'll live in `.sql()` escape hatches).
- Your data fits in memory and you just want fast local manipulation — Polars or pandas directly is simpler.
- You need long-term API stability with rare breaking changes; Ibis moves fast.

## Alternatives

- pola-rs/polars — fast single-engine local/streaming dataframe with its own execution; use it when you only need one fast in-process engine and don't need warehouse pushdown.
- duckdb/duckdb — the engine Ibis defaults to locally; use it directly when DuckDB is your only target and you'd rather write SQL.
- pandas-dev/pandas — the ubiquitous eager in-memory dataframe; use it when data fits in RAM and library interop matters more than portability.
- tobymao/sqlglot — the SQL parser/transpiler Ibis compiles through; use it directly when you only need SQL translation, not a dataframe API.
- apache/arrow — columnar in-memory format and compute (pyarrow); a lower-level layer, not a query-authoring API.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015 | Created by Wes McKinney at Cloudera as a Python front end for Impala/SQL-on-Hadoop. |
| 2.0 | 2020 | Modernized codebase; renewed development under Ursa/Voltron Data sponsorship. |
| 3.0 | 2022 | Internal reorganization of the expression/operation layers. |
| 4.0 | 2023 | Backends decoupled into extras; DuckDB became the default local backend. |
| 5.x–7.x | 2023 | Rapid iteration; datatype system and API refinements. |
| 8.0 | 2024 | SQLGlot-based compiler replaces SQLAlchemy for SQL backends[^3][^4]. |
| 9.0 | 2024 | pandas/Dask backends deprecated; further API changes[^3]. |
| 10.0 | 2025 | pandas/Dask backends removed; continued backend expansion[^3]. |

## References

[^1]: Ibis documentation, "Why Ibis?" https://ibis-project.org/why
[^2]: Ibis backends and per-backend operation support. https://ibis-project.org/backends/
[^3]: Ibis release notes. https://ibis-project.org/release_notes
[^4]: SQLGlot — the SQL transpiler Ibis compiles through. https://github.com/tobymao/sqlglot

## Tags

python, dataframe, sql, database, analytics, query-engine, duckdb, bigquery, data-engineering, backend-agnostic, lazy-evaluation, olap
