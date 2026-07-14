# dbt-labs/dbt-core

> SQL-first transformation framework: analysts write `select` statements, dbt compiles them into a dependency graph and materializes tables and views inside the warehouse.

[GitHub repo](https://github.com/dbt-labs/dbt-core) ·
[Official website](https://getdbt.com) ·
[License: Apache-2.0](https://github.com/dbt-labs/dbt-core/blob/main/LICENSE)

## Overview

dbt ("data build tool") is the de facto standard for the **T in ELT**: it does not move or process data itself, it generates SQL that the data warehouse runs. Analysts write models as `select` statements templated with Jinja; dbt resolves the `ref()`/`source()` calls into a DAG, compiles each model to raw SQL, and issues `CREATE TABLE`/`CREATE VIEW` (or incremental merge) statements against the warehouse. All compute stays in Snowflake, BigQuery, Redshift, Databricks, or Postgres — dbt is a code generator and orchestrator, not an execution engine[^1]. It began at Fishtown Analytics (renamed dbt Labs) around 2016 and reached a stable 1.0 in December 2021[^2].

The project's defining tension in 2026 is a **ground-up rewrite**. The `main` branch no longer hosts the Python implementation — that has moved to the [`1.latest`](https://github.com/dbt-labs/dbt-core/tree/1.latest) branch — and `main` is now dbt Core **v2.0 (alpha)**, written in Rust and serving as the foundation of the commercial "Fusion" engine[^3]. This is why GitHub reports the repo's primary language as Rust despite a decade of Python history. v2.0 promises much faster parse/compile times and a stricter language specification that rejects at parse time constructs the Python engine tolerated.

The second tension is commercial. dbt Core is Apache-2.0 and genuinely open, but dbt Labs monetizes through the dbt platform (formerly dbt Cloud), the semantic layer, and the Fusion engine — and not every Fusion component ships under Apache-2.0. Teams adopting dbt should separate "the open CLI" from "the hosted product" when evaluating lock-in.

## Getting Started

The Python distribution (v1, `1.latest`) installs per warehouse adapter:

```bash
pip install dbt-core dbt-snowflake   # or dbt-bigquery, dbt-postgres, dbt-databricks, ...
dbt init my_project
```

```sql
-- models/staging/stg_orders.sql
select order_id, customer_id, order_date, amount
from {{ source('shop', 'raw_orders') }}
where amount > 0
```

```sql
-- models/marts/customer_ltv.sql
{{ config(materialized='table') }}

select
    customer_id,
    count(*)   as num_orders,
    sum(amount) as lifetime_value
from {{ ref('stg_orders') }}   -- dbt wires the DAG edge from this ref()
group by 1
```

```bash
dbt run     # build models in dependency order
dbt test    # run schema/data tests
dbt build   # run + test + snapshot + seed in one DAG pass
```

The v2.0 / Fusion distribution ships instead as a single self-contained binary with no Python runtime required.

## Architecture / How It Works

dbt is a **compiler with a plugin backend**. The pipeline:

1. **Parse** — read every `.sql`/`.yml` file, render Jinja, and extract `ref()`/`source()`/`config()` calls to build a `manifest.json` and a DAG. On large Python-engine projects this parse step is the dominant latency and the primary motivation for the Rust rewrite.
2. **Compile** — expand Jinja and materialization macros into concrete warehouse SQL, written to `target/compiled/`.
3. **Run** — the **adapter** (a separate package, e.g. `dbt-snowflake`) translates a materialization into warehouse-specific DDL/DML and submits it over the warehouse's own connector. dbt never sees the rows.

**Materializations** are the core abstraction: `view` (default), `table` (full rebuild), `incremental` (append/merge only new rows via a `unique_key` and an `is_incremental()` predicate), and `ephemeral` (inlined as a CTE, never persisted). They are themselves Jinja macros, so the strategy is overridable per project.

**Adapters** are the coupling story. `dbt-core` and `dbt-adapters` were decoupled into separate release lines so warehouse vendors can ship connectors independently; the tradeoff is that a `dbt-core` version and its adapter versions must be kept compatible, and a warehouse-specific SQL dialect quirk becomes the adapter maintainer's problem, not core's. Tests, snapshots (SCD Type-2), seeds (CSV loads), sources with freshness checks, and the MetricFlow-based semantic layer (added after the 2023 Transform acquisition) all layer on top of this parse→compile→run core.

## Production Notes

**dbt does not bill you — your warehouse does.** The most common cost incident is an `incremental` model that silently falls back to a full-table scan (schema drift, a `full-refresh`, or a `unique_key` that isn't actually unique). Read `target/compiled/` to see the real SQL before blaming dbt.

**Slim CI depends on state.** `dbt build --select state:modified+` only rebuilds changed models and their children, but it requires the production `manifest.json` as an artifact to diff against. Wiring that artifact through CI is a setup step teams routinely get wrong, producing either full rebuilds or stale skips.

**Parse time scales poorly on the Python engine.** Projects with thousands of models see multi-minute parse times before a single model runs; this is the pain the Rust v2.0 rewrite targets. Do not adopt v2.0/Fusion for critical pipelines yet — it is alpha, the on-disk artifact formats may change, and its stricter parser will reject valid-in-v1 SQL, so migration is not a drop-in.

**Jinja debugging is opaque.** A templating error surfaces as a stack trace over generated SQL, not your source. `dbt compile` plus reading the compiled output is the standard debugging loop.

**Licensing due diligence.** dbt Core stays Apache-2.0, but if you standardize on the Fusion engine or the hosted dbt platform, confirm the license of each component you depend on — some Fusion pieces are not Apache-2.0. Treat "dbt Core" and "the dbt platform" as separately-governed.

## When to Use / When Not

**Use when:**
- Your transformation logic is SQL and your data already lives in a columnar warehouse (ELT, not ETL).
- You want software-engineering practices for analytics: version control, testing, DAG lineage, CI, environments.
- Multiple analysts need to build on each other's models without hand-managing execution order.

**Avoid when:**
- Your transforms are not expressible in warehouse SQL (heavy Python/ML feature engineering, non-tabular data) — reach for Spark/Python instead.
- You have no warehouse and no appetite to run one; dbt has nothing to compile against.
- You need row-level streaming/near-real-time; dbt is batch-oriented (microbatch incremental helps but is still batch).
- You want a single vendor-neutral tool and are wary of the Fusion/platform commercial gravity.

## Alternatives

- TobikoData/sqlmesh — direct competitor; use when you want native SQL parsing (no Jinja), column-level lineage, and virtual data environments with stateful diffs.
- dlt-hub/dlt — use for the **EL** (ingestion) step; complements rather than replaces dbt's T.
- apache/airflow — orchestration, not transformation; use to schedule dbt runs, not to write models.
- dataform-co/dataform — dbt-like SQL transformation tuned for BigQuery; use when all-in on Google Cloud.
- apache/spark — use when transformation exceeds warehouse SQL (large-scale Python, ML, non-tabular).

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2016 | Initial release at Fishtown Analytics; Jinja+SQL compilation model established. |
| 1.0 | 2021-12 | First stable major; `manifest.json` v4, standardized adapter contract[^2]. |
| 1.5 | 2023 | Model contracts, `access`/`groups`, `versions` — dbt Mesh foundations. |
| 1.6 | 2023 | MetricFlow semantic layer (post-Transform acquisition); cross-project `ref`. |
| 1.8 | 2024 | `dbt-core`/`dbt-adapters` decoupled into separate release lines; unit tests. |
| 1.9 | 2024 | `microbatch` incremental strategy. |
| 2.0-alpha | 2025–2026 | Rust rewrite on `main`; single binary, Parquet artifacts, Fusion foundation[^3]. |

## References

[^1]: dbt Labs, "What is dbt?" / product documentation. https://docs.getdbt.com/docs/introduction
[^2]: dbt Labs, "dbt Core v1.0 is here" — December 2021. https://www.getdbt.com/blog/dbt-core-v1-0-is-here
[^3]: dbt-labs/dbt-core README, "About dbt Core v2.0" and the Fusion engine. https://github.com/dbt-labs/dbt-core/blob/main/README.md · https://docs.getdbt.com/docs/fusion/about-fusion

## Tags

sql, data-transformation, elt, analytics-engineering, data-warehouse, jinja, rust, python, data-modeling, etl, dag
