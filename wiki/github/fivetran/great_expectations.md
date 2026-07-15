# fivetran/great_expectations

> Declarative data-quality testing for Python — assert what your data should look like, validate it against pandas, Spark, or SQL, and get rendered documentation of the results.

[GitHub repo](https://github.com/fivetran/great_expectations) ·
[Official website](https://docs.greatexpectations.io/) ·
[License: Apache-2.0](https://github.com/fivetran/great_expectations/blob/develop/LICENSE)

## Overview

Great Expectations (GX) is a Python framework for validating, documenting, and profiling data. The central abstraction is the **Expectation**: a declarative, named assertion about data — `ExpectColumnValuesToNotBeNull`, `ExpectColumnMeanToBeBetween`, `ExpectTableRowCountToBeBetween` — that reads as a specification and executes as a test. Groups of expectations form suites, suites run against batches of data, and results render into browsable HTML "Data Docs". The pitch is that data-quality rules should be shared, versioned, and human-readable rather than buried in ad-hoc `assert` statements or scattered SQL[^1].

The project began in 2017 and for years carried a reputation for power paired with a steep configuration surface: a stateful `great_expectations.yml`, a `DataContext`, batch-kwargs/batch-request plumbing, and multiple incompatible API generations (V2 "batch kwargs", V3 "batch request", then the 0.15-era "fluent datasources"). **GX 1.0**, released in 2024, was a ground-up API redesign that collapsed much of that ceremony into a smaller fluent surface — at the cost of a hard, non-transparent migration from the 0.18.x line[^2]. The repository now lives under the `fivetran/` organization following Fivetran's acquisition of Great Expectations; GX Core remains Apache-2.0 open source while the commercial roadmap centers on the hosted **GX Cloud** product. With ~11.6k stars, active daily commits, and a low open-issue count (mid-40s), the OSS core is maintained but visibly steered toward the SaaS offering.

The defining tension: GX buys you a rich, backend-portable expectation vocabulary and free documentation, but you pay for it in a stateful context, a per-backend capability matrix, and dependency weight that makes it heavy for small jobs.

## Getting Started

```bash
pip install great_expectations   # supports Python 3.10–3.13
```

```python
import great_expectations as gx
import pandas as pd

context = gx.get_context()                       # ephemeral in-memory context

df = pd.read_csv("orders.csv")

# connect the data: source -> asset -> batch definition -> batch
source = context.data_sources.add_pandas("orders_source")
asset = source.add_dataframe_asset(name="orders")
batch_def = asset.add_batch_definition_whole_dataframe("all")
batch = batch_def.get_batch(batch_parameters={"dataframe": df})

# assert an expectation against the batch
result = batch.validate(
    gx.expectations.ExpectColumnValuesToNotBeNull(column="user_id")
)
print(result.success)   # True / False
```

`get_context()` with no arguments returns an ephemeral context; pointing it at a project directory (or GX Cloud credentials) gives you a persisted `FileDataContext` or `CloudDataContext` that stores suites, checkpoints, and Data Docs.

## Architecture / How It Works

GX is a small number of composable objects layered over pluggable execution engines:

- **Data Context** — the entry point and configuration root. Three flavors: *Ephemeral* (in memory, nothing persisted), *File* (`great_expectations.yml` + a directory of suites/checkpoints/docs on disk), and *Cloud* (backed by GX Cloud). Almost every workflow starts from a context.
- **Data Sources → Data Assets → Batch Definitions → Batches** — the connection layer. A batch is the concrete unit of data an expectation runs against. The same asset can be sliced into batches by whole-dataframe, by table, or by time-based partitions.
- **Expectations & Expectation Suites** — expectations are declarative objects; a suite is a named, serializable collection of them. Suites are the reusable, version-controllable artifact.
- **Validation Definitions & Checkpoints** — a checkpoint binds a suite to a batch, runs it, produces **Validation Results**, and fires **Actions** (update Data Docs, send Slack/email, raise on failure). Checkpoints are how GX runs inside a pipeline (Airflow, Dagster, Prefect, cron).
- **Data Docs** — static HTML rendering of suites and their latest validation results. This is a genuine differentiator: the documentation is a byproduct of validation, not a separate maintenance task.

The internal mechanism worth understanding is the **metrics system**. An expectation does not run SQL or pandas directly; it decomposes into metrics (a column's null count, a distinct-value set, a mean) that each **execution engine** — Pandas, Spark (via PySpark), or SQL (via SQLAlchemy) — implements independently. This is what makes expectations backend-portable, and it is also the source of the most important caveat: an expectation only works on a backend if its underlying metrics are implemented there. Coverage is broadest on pandas and thinnest on some SQL dialects, so a suite that passes against a pandas sample can raise `NotImplementedError` against the production warehouse.

## Production Notes

**The 1.0 migration is a wall, not a ramp.** The pre-1.0 (0.18.x) config format, datasource definitions, and checkpoint YAML do not carry forward cleanly, and there is no fully automatic upgrade path. Many teams remained pinned to `great_expectations==0.18.x` well after 1.0 shipped. Treat a 0.18 → 1.0 move as a rewrite of your GX layer, not a version bump[^2].

**Where computation happens matters.** On Spark and SQL backends, GX pushes work down to the engine — good, but each expectation can translate into its own query. A large suite against a SQL source can mean dozens of separate round-trips unless expectations share metrics. On the pandas backend, GX loads the whole batch into memory; "validate a 40 GB table with pandas" is a common footgun. Partition into batches or use a Spark/SQL source for anything large.

**State and sprawl.** A File context accumulates suites, checkpoints, validation history, and generated Data Docs on disk (or S3/GCS/Azure). Data Docs in particular grow unbounded — every validation run adds artifacts — and need lifecycle management on object stores. In CI, prefer an ephemeral context and build suites in code to avoid committing a stateful `great_expectations/` directory.

**Dependency weight.** GX pulls in a substantial dependency tree (SQLAlchemy, marshmallow/pydantic, rendering stack, and the backend driver you choose). For a single dataframe check this is heavy; for a platform-wide data-quality gate it amortizes.

**Backend capability matrix is real.** Before standardizing on an expectation, confirm it is implemented for your production engine and dialect. The compatibility reference documents supported data sources; assume the pandas experience is the most complete and validate against your real backend early[^3].

**Vendor direction.** Post-acquisition, active feature investment leans toward GX Cloud. The OSS Core is Apache-2.0 and maintained, but if you need managed alerting, hosted Data Docs, and a UI, that is deliberately the paid surface — factor the roadmap into a long-horizon bet.

## When to Use / When Not

**Use when:**
- You want data-quality rules that are declarative, shareable across a team, and readable by non-authors.
- You need the same expectation vocabulary to run against pandas locally and Spark/SQL in production.
- Auto-generated, browsable documentation of data quality (Data Docs) has real value for your stakeholders.
- You're gating a pipeline (Airflow/Dagster/Prefect) and want validation results with pluggable failure actions.

**Avoid when:**
- You just need lightweight in-code dataframe schema checks — GX's context/suite/checkpoint machinery is overkill; reach for pandera.
- Your transformations already live in dbt and simple column tests cover you.
- You're on a niche SQL dialect where the expectations you need aren't implemented.
- You want a zero-state, single-function `validate(df, schema)` call with no project scaffolding.

## Alternatives

- unionai-oss/pandera — use instead when you want lightweight, decorator/schema-based dataframe validation embedded directly in Python code without GX's config surface.
- sodadata/soda-core — use instead for SQL-first data quality expressed as concise SodaCL YAML checks with a leaner setup.
- dbt-labs/dbt-core — use instead when your models already live in dbt; built-in tests plus the dbt-expectations package cover many GX-style assertions.
- awslabs/deequ — use instead for JVM/Spark-native data profiling and constraint verification at large scale (python-deequ provides Python bindings).
- frictionlessdata/frictionless-py — use instead when you need Table Schema / Data Package validation for file-based tabular data interchange.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-09-11 | First public release; Expectations + Data Docs concept[^1]. |
| V2 API | ~2019 | "Batch kwargs" datasource/validation API. |
| V3 API | ~2020 | "Batch request" API; execution-engine abstraction. |
| 0.15.x | ~2022 | Fluent datasources introduced alongside the older API. |
| 0.18.x | ~2023 | Last major pre-1.0 line; widely pinned during 1.0 migration. |
| 1.0 | 2024 | Ground-up API redesign; smaller fluent surface, hard migration[^2]. |
| — | (post-1.0) | Repo transferred to `fivetran/` after Fivetran acquired GX; focus shifts to GX Cloud. |

## References

[^1]: Great Expectations documentation — "Introduction to GX Core". https://docs.greatexpectations.io/docs/core/introduction/
[^2]: Great Expectations — GX 1.0 announcement and migration guidance. https://greatexpectations.io/blog/the-fastest-way-to-get-started-with-gx-v1-0-is-here
[^3]: Great Expectations — compatibility / integration support reference. https://docs.greatexpectations.io/docs/application_integration_support

## Tags

python, data-quality, data-validation, data-engineering, testing, pandas, spark, sql, mlops, data-pipelines, data-profiling, etl
