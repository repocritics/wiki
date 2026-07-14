# dagster-io/dagster

> An asset-oriented data orchestrator: you declare the tables, models, and reports you want to exist, and Dagster schedules the code that keeps them fresh.

[GitHub repo](https://github.com/dagster-io/dagster) ·
[Official website](https://dagster.io) ·
[License: Apache-2.0](https://github.com/dagster-io/dagster/blob/master/LICENSE)

## Overview

Dagster is a Python data-pipeline orchestrator started in 2018 by Nick Schrock (a
co-creator of GraphQL) under the company Elementl, later renamed Dagster Labs[^1]. It
competes directly with Apache Airflow, but around a different central abstraction. Where
Airflow orchestrates *tasks* (run this operator, then that one), Dagster's headline model
is the **software-defined asset**: you annotate a Python function with `@dg.asset`, declare
its dependencies by function argument name, and Dagster infers the graph, tracks lineage,
and materializes assets on schedule or on demand[^2].

The defining tension is exactly that asset framing. It is a genuinely better fit for
analytics and ML workloads whose *output* (a warehouse table, a trained model) is the thing
you care about, and it gives you data lineage, freshness, and cataloging almost for free.
But it is a heavier conceptual load than "run these scripts in order," and teams migrating
from Airflow routinely underestimate the relearning. Dagster also supports the older
task-style API (`@op` / `@job`) for pipelines that genuinely are side-effect sequences, so
you inherit two overlapping mental models in the same codebase.

Dagster Labs funds the project by selling Dagster+ (formerly Dagster Cloud), a hosted
control plane; the orchestration engine itself is Apache-2.0 and fully self-hostable[^3].

## Getting Started

Dagster is on PyPI and supports Python 3.9–3.14.

```bash
uv add dagster dagster-webserver dagster-dg-cli
```

```python
# defs.py
import dagster as dg
import pandas as pd

@dg.asset
def raw_users() -> pd.DataFrame:
    return pd.read_csv("https://example.com/users.csv")

@dg.asset
def active_users(raw_users: pd.DataFrame) -> pd.DataFrame:
    # dependency is expressed by the parameter name matching the asset above
    return raw_users[raw_users["last_seen_days"] < 30]
```

```bash
dagster dev -f defs.py   # local webserver + daemon at http://localhost:3000
```

The parameter `raw_users` on `active_users` *is* the dependency edge — there is no explicit
`>>` wiring. Click "Materialize all" in the UI and both assets run in dependency order.

## Architecture / How It Works

The system is several long-lived processes plus your code, coordinated through a metadata
database:

- **Definitions / code locations.** Your assets, jobs, schedules, sensors, and resources
  are collected into a `Definitions` object. Dagster loads your code in a *separate* process
  (historically a gRPC server) so that a bad import in user code cannot crash the control
  plane, and so the UI can introspect definitions without executing them.
- **The instance (`DagsterInstance`).** All run history, the event log, schedule/sensor
  state, and asset materialization records live in storage backing the instance. It defaults
  to SQLite under `$DAGSTER_HOME`; production uses PostgreSQL (or MySQL)[^4].
- **The webserver (`dagster-webserver`, historically Dagit).** A React UI backed by a
  GraphQL API. It is read/introspect plus launch; it does not itself execute runs.
- **The daemon (`dagster-daemon`).** A background process that ticks schedules, evaluates
  sensors, runs the run-queue coordinator, and drives auto-materialization/declarative
  automation. **If the daemon is not running, schedules and sensors silently do nothing** —
  this is the single most common "why isn't my pipeline firing" cause.
- **Executors and run launchers.** A *run launcher* decides where a run's process starts
  (local subprocess, a Kubernetes Job, ECS task, Docker container); an *executor* decides how
  the steps *within* a run parallelize (in-process, multiprocess, or per-step k8s pods).

Between assets, data is moved by **IO managers** — a pluggable layer that decides where a
function's return value is persisted (local files, S3, a Snowflake table) and how the next
asset reads it. This is powerful but is also where newcomers get surprised: returning a
DataFrame does not "just save it somewhere" unless an IO manager says so.

## Production Notes

- **You must run the daemon.** `dagster dev` bundles webserver + daemon for convenience, but
  in production they are separate deployments. Forgetting the daemon means no schedules, no
  sensors, no run-queue throttling.
- **Version lockstep is mandatory.** `dagster` and every `dagster-*` integration package
  (dagster-aws, dagster-dbt, dagster-k8s, …) must be pinned to the *same* version. Mixed
  versions produce confusing import/serialization errors. Pin them together in one bump.
- **Schema migrations on upgrade.** Minor-version upgrades can change the instance storage
  schema; run `dagster instance migrate` against your PostgreSQL after upgrading, and expect
  a maintenance window on large event logs.
- **The event log is the scaling pressure point.** Every step, materialization, and asset
  check writes events. High-frequency or highly-partitioned pipelines make the event-log and
  asset-materialization tables grow fast; budget for PostgreSQL sizing and periodic
  wipe/retention on run history.
- **Partitions are strong but sharp.** Partitioned assets (e.g. daily) are a real strength,
  but large multi-dimensional partition sets and full backfills can generate enormous numbers
  of runs and heavy UI/database load. Model partition granularity deliberately.
- **Kubernetes is the reference production topology.** The official Helm chart deploys the
  webserver, daemon, and per-code-location gRPC servers; the `K8sRunLauncher` launches each
  run as its own Job[^5]. Self-hosting outside k8s is possible but less paved.
- **The `dg` CLI / Components layer is newer.** The scaffolding CLI (`dagster-dg-cli`) and
  the Components system are the current recommended project shape but have iterated quickly;
  older tutorials and `@repository`-era code you find online may not match current APIs.

## When to Use / When Not

**Use when:**
- Your pipelines produce durable data *assets* (warehouse tables, ML models, reports) and you
  want lineage, freshness, and a catalog as first-class concepts.
- You want strong local development, unit-testable pipelines, and typed inputs/outputs.
- You use dbt, Spark, or a modern warehouse and want native integration and asset-level
  observability across them.

**Avoid when:**
- You mostly need generic task DAGs of arbitrary shell/side-effect steps with a huge existing
  operator ecosystem — Airflow's breadth and community may fit better.
- You want a tiny dependency-light scheduler; Dagster is a full platform with a database,
  daemon, and UI to operate.
- Your team cannot invest in learning the asset model and would treat it as "Airflow with
  extra steps."

## Alternatives

- apache/airflow — the incumbent task-based orchestrator; choose it for maximum operator/
  ecosystem breadth and existing team familiarity.
- PrefectHQ/prefect — Pythonic flow orchestration with a lighter, more imperative model; use
  it when you want dynamic task graphs without Dagster's asset framing.
- flyteorg/flyte — strongly-typed, Kubernetes-native ML/data pipelines; use it when
  container-per-task reproducibility and ML workflows dominate.
- kestra-io/kestra — YAML-declared, language-agnostic orchestration on the JVM; use it when
  you want pipelines defined in config rather than Python.
- mage-ai/mage-ai — notebook-style pipeline builder; use it for a lower-ceremony, more
  interactive authoring experience.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo public | 2018-04 | Project open-sourced by Elementl[^1]. |
| 0.6 | 2020 | Dagit UI, solids/pipelines era, gRPC code servers. |
| 0.15 | 2022-05 | Software-defined assets promoted toward the core model[^2]. |
| 1.0 | 2022-08 | Stable API commitment; assets as the headline abstraction[^6]. |
| 1.5 | 2023 | Dagit renamed Dagster UI (`dagster-webserver`); auto-materialize policies. |
| — | 2024 | Dagster Cloud rebranded Dagster+; Declarative Automation[^3]. |
| — | 2025 | `dg` CLI and Components scaffolding become the recommended project shape. |

## References

[^1]: Dagster Labs (formerly Elementl) — company/history. https://dagster.io/about
[^2]: Dagster docs, "Assets" (software-defined assets). https://docs.dagster.io/guides/build/assets
[^3]: Dagster+ product (hosted control plane). https://dagster.io/plus
[^4]: Dagster docs, "Dagster instance" (storage, `dagster.yaml`). https://docs.dagster.io/guides/deploy/dagster-instance-configuration
[^5]: Dagster docs, "Deploying with Helm on Kubernetes". https://docs.dagster.io/guides/deploy/deployment-options/kubernetes
[^6]: Dagster blog, "Dagster 1.0" — 2022-08. https://dagster.io/blog/dagster-1-0

## Tags

python, data-orchestration, data-engineering, etl, data-pipelines, software-defined-assets, mlops, workflow-automation, scheduler, data-lineage
