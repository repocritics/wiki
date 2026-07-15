# kedro-org/kedro

> A Python framework for structuring data science and data engineering code into reproducible, modular pipelines — an opinionated project template plus a data-I/O abstraction, not a scheduler.

[GitHub repo](https://github.com/kedro-org/kedro) ·
[Official website](https://kedro.org) ·
[License: Apache-2.0](https://github.com/kedro-org/kedro/blob/main/LICENSE.md)

## Overview

Kedro is a Python framework that imposes software-engineering discipline on data science code: a standard project layout, a declarative data-I/O layer (the Data Catalog), and pipelines expressed as directed graphs of pure Python functions. It originated inside QuantumBlack, McKinsey's analytics arm, and was open-sourced in 2019[^1]; the project is now hosted by the LF AI & Data Foundation[^2]. As of 2026 it has ~10.9k stars and ~1.1k forks, with releases shipping roughly monthly — an actively maintained project past its long-awaited 1.0.

The defining idea is separation of concerns between *what your code computes* and *where the data lives*. Node functions receive and return plain Python objects (DataFrames, models, arrays) and never touch file paths, credentials, or formats. All I/O is declared in `catalog.yml`, so the same pipeline runs against a local CSV in development and a partitioned Parquet dataset on S3 in production with no code change. This is Kedro's strongest selling point and the source of its main friction: the abstraction is worth it on team projects with real deployment targets, and pure overhead on a one-off notebook analysis.

Kedro is frequently misread as a workflow orchestrator like Airflow or Dagster. It is not. Kedro *authors and runs* pipelines in-process; it has no scheduler, no UI for triggering runs, no retry/backfill machinery. For scheduled or distributed execution you deploy a Kedro pipeline onto an orchestrator (Airflow, Prefect, Argo, Kubeflow, Databricks) via a plugin[^3]. Understanding this boundary is the single most important thing before adopting it.

## Getting Started

```bash
uv pip install kedro        # or: conda install -c conda-forge kedro
kedro new                   # scaffolds a project from a cookiecutter template
cd my-project
kedro run
```

Declare data in `conf/base/catalog.yml` — code never sees a filepath:

```yaml
raw_reviews:
  type: pandas.CSVDataset
  filepath: data/01_raw/reviews.csv

model_input:
  type: pandas.ParquetDataset
  filepath: data/05_model_input/model_input.parquet
```

Define a pipeline as nodes wiring catalog entries and parameters together:

```python
# src/my_project/pipelines/data_processing/pipeline.py
from kedro.pipeline import Pipeline, node, pipeline

def clean(reviews):
    return reviews.dropna()

def create_pipeline() -> Pipeline:
    return pipeline([
        node(
            func=clean,
            inputs="raw_reviews",       # resolved from the Data Catalog
            outputs="model_input",      # persisted per its catalog entry
            name="clean_reviews",
        ),
    ])
```

## Architecture / How It Works

Kedro has four core abstractions:

1. **Node** — a pure Python function wrapped with declared input/output names. Nodes carry no I/O and no side effects by contract.
2. **Pipeline** — a set of nodes. Kedro resolves execution order automatically by topological sort of the input/output name graph; you never declare edges explicitly, they are inferred from matching names.
3. **DataCatalog** — the I/O registry. Each named entry maps to a *dataset* class (e.g. `pandas.CSVDataset`, `spark.SparkDataset`, `pickle.PickleDataset`) implementing a `load()`/`save()` interface over `fsspec`, so local, S3, GCS, ABFS, and HDFS paths work through the same config.
4. **Runner** — executes the resolved graph. `SequentialRunner` (default), `ThreadRunner`, and `ParallelRunner` (multiprocessing) ship in core; the runner is where the pure-function contract pays off, since independent nodes can run concurrently.

**Datasets live in a separate package.** Core Kedro ships only the framework; the connector library is `kedro-datasets`, developed in the `kedro-plugins` monorepo and versioned independently[^4]. This split (completed in the 0.19 line) means the core has few dependencies, but you install and pin dataset support separately.

**Configuration** is layered by environment. `conf/base/` holds shared config; `conf/local/` overrides it with machine-specific values and secrets (git-ignored by default). The `OmegaConfigLoader` (default since 0.19) resolves them with OmegaConf, giving variable interpolation, templating, and runtime overrides.

**Session and context.** A `KedroSession` is created per run; it builds a `KedroContext` that assembles the catalog, parameters, and pipelines. **Hooks** (built on `pluggy`, the same plugin system as pytest) let you inject behavior at defined lifecycle points — `before_node_run`, `after_catalog_created`, `on_pipeline_error`, etc. This is the extension seam used by experiment trackers, MLflow integrations, and custom logging.

The data folder follows a numbered layered convention (`01_raw` → `08_reporting`) that encodes a data-engineering discipline. It is a convention, not enforced by code.

## Production Notes

**Kedro is not the runtime you deploy.** In production you convert the pipeline into an orchestrator's DAG. `kedro-airflow` emits an Airflow DAG, `kedro-databricks` targets Databricks jobs, and community plugins cover Argo, Kubeflow, Prefect, and SageMaker. Each translation has fidelity gaps: node-to-task granularity, how intermediate datasets are persisted between distributed tasks (in-memory datasets do *not* survive across tasks on separate workers — every cross-task dataset must be a persisted catalog entry), and how parameters/credentials are injected.

**MemoryDataset is the classic footgun.** Any pipeline output not declared in the catalog defaults to a `MemoryDataset` — fine in a single-process `kedro run`, but silently lost when the same pipeline is fanned out across distributed workers. Distributed deployments require every inter-node dataset to be catalog-backed on shared storage.

**Spark integration is manual.** Kedro does not manage a SparkSession. You create it in a hook or `spark.yml`, and use `spark.SparkDataset`; lazy DataFrames and Kedro's eager save/load model interact in ways that need care to avoid recomputation.

**Versioning is copy-based.** Enabling `versioned: true` on a dataset writes each run's output to a timestamped path. This is per-dataset file versioning, not a content-addressed store like DVC or lakeFS — it grows storage linearly and has no dedup or diff.

**Upgrade pains.**
- **0.17 → 0.18** introduced `KedroSession` and reworked the CLI/project structure; a substantial migration.
- **0.18 → 0.19** removed `kedro.extras.datasets` (moved to the standalone `kedro-datasets` package) and renamed every dataset class from the `DataSet` suffix to `Dataset` (e.g. `CSVDataSet` → `CSVDataset`)[^5] — a wide, mechanical but breaking rename.
- **0.19 → 1.0** (2025-07) removed long-deprecated APIs and committed to semantic versioning; 1.x is the first line with a stability guarantee[^6].

**Testing.** Because nodes are pure functions, they unit-test without Kedro machinery — call them with plain inputs. Testing the wiring is separate from testing the logic, which is the intended payoff of the design.

## When to Use / When Not

**Use when:**
- A team of mixed software-engineering experience needs a shared, enforced project structure.
- The same pipeline must run against different data backends across dev/staging/prod without code edits.
- You are moving analysis out of notebooks toward maintainable, testable, deployable code.
- You want pipeline visualization and lineage (via Kedro-Viz) and plan to deploy onto an existing orchestrator.

**Avoid when:**
- You need a scheduler with triggers, retries, backfills, and observability — reach for Dagster, Prefect, or Airflow directly.
- The work is exploratory one-off analysis; the catalog/config ceremony is pure overhead.
- Your pipeline is dynamic at runtime (graph shape depends on data) — Kedro's DAG is resolved statically before execution.
- You want asset-and-lineage as first-class runtime concepts rather than a project convention.

## Alternatives

- dagster-io/dagster — use instead when you want assets, scheduling, retries, and a run UI as first-class runtime features, not just authoring structure.
- PrefectHQ/prefect — use instead when you want dynamic, Pythonic workflows with a managed scheduler and less imposed project layout.
- apache/airflow — use instead when scheduled orchestration of heterogeneous tasks is the primary need (Kedro often deploys onto it, not against it).
- ploomber/ploomber — use instead when you want notebook-native pipelines with lighter structural overhead.
- dbt-labs/dbt-core — use instead when your transformations live in SQL inside a warehouse rather than in Python.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.15 | 2019 | Open-sourced by QuantumBlack (McKinsey)[^1]. |
| 0.17.0 | 2020-12 | `KedroSession`, reworked project structure. |
| 0.18.0 | 2022-05 | New CLI, hooks refinements, config overhaul. |
| 0.19.0 | 2023-12 | Datasets split to `kedro-datasets`; `DataSet`→`Dataset` rename; `OmegaConfigLoader` default[^5]. |
| 1.0.0 | 2025-07-22 | First stable major; semantic-versioning commitment[^6]. |
| 1.5.0 | 2026-06-29 | Latest minor in the 1.x line. |

## References

[^1]: QuantumBlack, "Kedro is now open source" — 2019. https://medium.com/quantumblack/introducing-kedro-the-open-source-library-for-production-ready-machine-learning-code-d1c6d26ce2cf
[^2]: LF AI & Data Foundation — Kedro project page. https://lfaidata.foundation/projects/kedro/
[^3]: Kedro docs, "Deployment". https://docs.kedro.org/en/stable/deployment/
[^4]: `kedro-plugins` monorepo (home of `kedro-datasets`). https://github.com/kedro-org/kedro-plugins
[^5]: Kedro 0.19.0 release notes / migration guide. https://github.com/kedro-org/kedro/blob/main/RELEASE.md
[^6]: Kedro 1.0.0 release — 2025-07-22. https://github.com/kedro-org/kedro/releases/tag/1.0.0

## Tags

python, data-engineering, data-science, mlops, pipelines, dag, machine-learning, data-catalog, reproducibility, workflow-framework
