# SQLMesh/sqlmesh

> A SQL transformation framework that parses your SQL instead of templating strings — dbt-compatible, with virtual data environments and semantic-diff-driven deploys.

[GitHub repo](https://github.com/SQLMesh/sqlmesh) ·
[Official website](https://sqlmesh.readthedocs.io/en/stable/) ·
[License: Apache-2.0](https://github.com/SQLMesh/sqlmesh/blob/main/LICENSE)

## Overview

SQLMesh is a data transformation framework built by Tobiko Data and, since 2024, hosted under the Linux Foundation[^1]. It occupies the same slot as dbt — you write SQL models, it builds tables/views in your warehouse in dependency order — but takes a fundamentally different technical bet: instead of treating SQL as opaque templated text, SQLMesh parses and understands it. That parsing comes from SQLGlot[^2], a pure-Python SQL parser and transpiler by the same author (Toby Mao), which gives SQLMesh column-level lineage, cross-dialect transpilation, and the ability to tell whether a change to a model is *breaking* (alters output) or *non-breaking* (cosmetic) before anything runs.

The defining feature is the **virtual data environment**. A `plan`/`apply` workflow (deliberately modeled on Terraform) computes a semantic diff of your models, shows which downstream tables are affected, and lets you materialize a full isolated dev environment that shares unchanged physical tables with production through view swapping — so spinning up a dev branch usually costs no warehouse compute for the parts you didn't touch. Promotion to production is then a metadata operation (repointing views), giving blue-green-style deploys rather than a full rebuild[^3].

The central tradeoff: SQLMesh is more correct and more efficient than the Jinja-string approach, but it is more machinery. It maintains a **state database** of model snapshots and fingerprints, introduces concepts (snapshots, virtual vs physical layers, forward-only changes, restatements) that have a real learning curve, and its ecosystem and community are a fraction of dbt's size. It is backwards-compatible with dbt projects, which is the intended on-ramp — but adopting the model fully means thinking in SQLMesh's terms, not dbt's.

## Getting Started

```bash
mkdir sqlmesh-example && cd sqlmesh-example
python -m venv .venv && source .venv/bin/activate
pip install 'sqlmesh[lsp]'   # base package; [lsp] adds the VSCode/editor extension
sqlmesh init duckdb          # scaffolds a project against a local DuckDB
```

A model is plain SQL with a `MODEL(...)` header — no Jinja or YAML required for the definition:

```sql
MODEL (
  name demo.stg_payments,
  kind INCREMENTAL_BY_TIME_RANGE (time_column payment_date),
  cron '@daily',
  grain payment_id,
  audits (NOT_NULL(columns = (payment_id)), UNIQUE_VALUES(columns = (payment_id)))
);

SELECT
  id           AS payment_id,
  order_id,
  amount / 100 AS amount,        -- cents -> dollars
  payment_date
FROM demo.seed_raw_payments
WHERE payment_date BETWEEN @start_ds AND @end_ds;   -- SQLMesh injects the window
```

```bash
sqlmesh plan            # semantic diff: what changed, what's breaking, what backfills
sqlmesh plan dev        # same, into an isolated virtual environment named `dev`
sqlmesh test            # run unit tests (fixture in/out, no warehouse round-trip)
sqlmesh audit           # run data-quality audits against materialized data
```

## Architecture / How It Works

The pipeline hangs off SQLGlot. Every model is parsed into an AST, which is what makes the rest possible:

- **Snapshots and fingerprinting.** Each model version is captured as a *snapshot* with a fingerprint derived from its parsed query plus its upstream dependencies. When you change a model, SQLMesh recomputes fingerprints and diffs ASTs to classify the change. A whitespace or comment edit produces no new data version; an added column may be non-breaking; a changed filter is breaking and triggers backfills of that model and its affected descendants only.
- **Physical vs virtual layer.** Data physically lands in fingerprint-suffixed tables in a physical schema. Environments (`prod`, `dev`, feature branches) are thin *virtual* layers of views pointing at those physical tables. Two environments sharing an unchanged model share one physical table. Promotion swaps view targets — no data movement.
- **Column-level lineage.** Because the query is parsed, SQLMesh resolves lineage down to individual columns, used both for impact analysis in `plan` and for the diffs surfaced in the CLI and VSCode extension.
- **Dialect transpilation.** You can author in one SQL dialect and SQLMesh transpiles to the target engine's dialect before execution, catching many syntax/semantics errors locally instead of in the warehouse.
- **State backend.** All of the above is tracked in a **state connection** — a database (often the warehouse itself, or a separate Postgres) holding snapshot metadata, environment records, and interval tracking for incremental models.

Model kinds encode materialization semantics explicitly: `FULL`, `VIEW`, `INCREMENTAL_BY_TIME_RANGE`, `INCREMENTAL_BY_UNIQUE_KEY`, `SCD_TYPE_2`, `EMBEDDED`, and seeds. Incremental models track which time intervals have been loaded, so reruns process only missing windows, and `restate` lets you surgically rebuild a range.

## Production Notes

- **The state connection is not optional.** SQLMesh needs a durable state database, and its performance/consistency matter. Pointing state at a slow or high-latency warehouse (e.g. a serverless data warehouse charged per query) makes `plan` sluggish and can rack up cost. A dedicated Postgres for state is a common production choice; back it up — losing state means losing SQLMesh's understanding of what's already been built.
- **Virtual environments depend on view-swap support.** The zero-copy dev environment story assumes the engine supports cheap views over the physical layer. Engines with weak or costly view semantics dilute the benefit.
- **Forward-only changes.** For large production tables you often mark changes `forward-only` to avoid full backfills, accepting that dev and prod computed the column differently for historical rows. Understanding when a change is safe as forward-only is an operator skill, not automatic.
- **Housekeeping.** Fingerprinted physical tables accumulate; SQLMesh has a `janitor` that garbage-collects snapshots past their TTL, but you should understand its retention behavior before trusting it in a cost-sensitive warehouse.
- **Scheduling.** SQLMesh ships a built-in scheduler and integrates with Airflow and Dagster; there is no managed control plane in the OSS project. Tobiko Cloud (`tcloud`) is the paid product that adds hosted scheduling, observability, and a UI — the open-source/commercial line runs along operations, not the core engine.
- **dbt migration is real but not free.** SQLMesh can run existing dbt projects via its dbt adapter, and many teams start there. Jinja-heavy dbt macros, exotic packages, and heavy reliance on dbt-specific behavior are the friction points; a straight lift usually works, extracting full value means rewriting toward SQLMesh-native models.
- **Maturity.** Actively developed with frequent releases and a responsive Slack community, but the surrounding ecosystem (packages, tutorials, hiring pool, third-party integrations) is far smaller than dbt's. Expect to read source and docs rather than Stack Overflow.

## When to Use / When Not

**Use when:**
- Your warehouse compute bill from dev/CI rebuilds is a real line item — virtual environments and change-aware backfills target exactly this.
- Correctness matters and you want breaking-vs-non-breaking classification and column-level lineage before deploying.
- You run many isolated dev/feature environments and want them cheap.
- You want true blue-green promotion rather than rebuild-and-swap.

**Avoid when:**
- Your project is small and dbt's simpler mental model plus its huge ecosystem outweigh SQLMesh's efficiency gains.
- You depend on a large surface of dbt packages/macros or need to hire from the broad dbt talent pool.
- You can't or won't operate a state backend and think in snapshots — the machinery is the cost of the benefits.
- You need a managed, UI-first experience without paying for Tobiko Cloud.

## Alternatives

- dbt-labs/dbt-core — the incumbent SQLMesh is compatible with; use it when the mature ecosystem, community, and hiring pool matter more than semantic understanding or virtual environments.
- tobikodata/sqlglot — the SQL parser/transpiler underneath SQLMesh; use it directly when you only need SQL parsing, linting, or cross-dialect transpilation, not a full framework.
- dagster-io/dagster — asset-based orchestrator; use it when you need general-purpose pipeline orchestration across many systems, not SQL-transformation-specific semantics.
- apache/airflow — general workflow scheduler; use it when arbitrary DAG scheduling is the primary need and transformation logic is secondary.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2022-09 | Repository created; project founded at Tobiko Data[^4]. |
| 0.x | 2023 | Public releases: virtual environments, plan/apply, dbt adapter, incremental kinds. |
| — | 2024 | Project contributed to / hosted under the Linux Foundation[^1]. |
| 0.x | 2024–2025 | VSCode extension + LSP, column-level lineage in editor, expanded engine support. |

*(SQLMesh remains on a pre-1.0 `0.x` line; consult the changelog for exact version dates.)*

## References

[^1]: SQLMesh README — "SQLMesh is a project of the Linux Foundation." https://github.com/SQLMesh/sqlmesh
[^2]: SQLGlot — SQL parser, transpiler, and optimizer that SQLMesh is built on. https://github.com/tobikodata/sqlglot
[^3]: SQLMesh docs, "Plans" and "Environments." https://sqlmesh.readthedocs.io/en/stable/concepts/plans/
[^4]: GitHub API — repository `created_at` 2022-09-23, license Apache-2.0. https://github.com/SQLMesh/sqlmesh

## Tags

python, sql, data-transformation, data-engineering, elt, dbt-alternative, dataops, sqlglot, incremental-models, data-warehouse
