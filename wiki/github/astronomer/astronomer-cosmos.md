# astronomer/astronomer-cosmos

> Renders a dbt project as native Airflow tasks, so each dbt model becomes an Airflow task with its own retries, alerting, and scheduling.

[GitHub repo](https://github.com/astronomer/astronomer-cosmos) ·
[Official website](https://astronomer.github.io/astronomer-cosmos/) ·
[License: Apache-2.0](https://github.com/astronomer/astronomer-cosmos/blob/main/LICENSE)

## Overview

Cosmos is a Python provider package, maintained by Astronomer, that turns a dbt Core (or, more recently, dbt Fusion) project into an Airflow DAG or Task Group without hand-writing an operator per model[^1]. Instead of running `dbt run` as one opaque `BashOperator`, Cosmos parses the project's dependency graph and emits one Airflow task per dbt node, so model-level retries, alerting, data-aware scheduling, and the Airflow UI's task view all work at dbt-model granularity.

The audience is teams that already run Airflow and already have a dbt project, and who want the two to share one control plane rather than treating dbt as a black box invoked from a single task. The alternative postures are the dbt Cloud scheduler (a separate SaaS control plane) or a coarse `BashOperator`/`KubernetesPodOperator` wrapping the whole `dbt build`.

The defining tension is graph duplication. Airflow wants to know the task graph at DAG-parse time; dbt only fully knows its graph after it has compiled the project into a `manifest.json`. Cosmos bridges this by either running `dbt ls`/parse at DAG-parse time or reading a pre-built manifest — and that choice (`LoadMode`) is the single biggest determinant of whether Cosmos is fast and stable or slow and flaky in production.

## Getting Started

```bash
pip install astronomer-cosmos
# extras select the profile mapping / execution backend, e.g.
pip install "astronomer-cosmos[dbt-postgres]"
```

```python
# dags/my_cosmos_dag.py
from datetime import datetime
from cosmos import DbtDag, ProjectConfig, ProfileConfig, ExecutionConfig
from cosmos.profiles import PostgresUserPasswordProfileMapping

profile_config = ProfileConfig(
    profile_name="jaffle_shop",
    target_name="dev",
    profile_mapping=PostgresUserPasswordProfileMapping(
        conn_id="postgres_default",          # a normal Airflow connection
        profile_args={"schema": "public"},
    ),
)

my_dag = DbtDag(
    project_config=ProjectConfig("/usr/local/airflow/dbt/jaffle_shop"),
    profile_config=profile_config,
    execution_config=ExecutionConfig(dbt_executable_path="/usr/local/airflow/dbt_venv/bin/dbt"),
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    dag_id="jaffle_shop",
)
```

`DbtDag` produces a whole DAG; `DbtTaskGroup` embeds the same rendered graph inside an existing DAG so dbt can sit between ingestion and downstream tasks.

## Architecture / How It Works

Cosmos is configured through four config objects that map cleanly onto dbt's own concepts:

- **`ProjectConfig`** — where the dbt project lives, and how its graph is discovered.
- **`ProfileConfig`** — how to build `profiles.yml`. The key feature is `profile_mapping`: instead of a checked-in `profiles.yml` with credentials, Cosmos renders one at runtime from an Airflow connection, so warehouse secrets live in Airflow's connection/secrets backend.
- **`ExecutionConfig`** — how dbt actually runs (`ExecutionMode`): `local`, `virtualenv`, `docker`, `kubernetes`, `aws_eks`, `azure_container_instance`, `gcp_cloud_run`, or `aws_ecs`.
- **`RenderConfig`** — how the graph is parsed at render time (`LoadMode`), plus node selection (`select`/`exclude` using dbt selector syntax).

The two axes that matter most are orthogonal: **how Cosmos learns the graph** (LoadMode) and **how each task executes dbt** (ExecutionMode).

`LoadMode` options: `dbt_ls` runs `dbt ls` during DAG parsing (accurate, but pays a dbt subprocess cost on every scheduler parse), `dbt_manifest` reads a pre-compiled `manifest.json` (fast, no dbt at parse time, but the manifest must be produced by CI/CD and kept in sync), `custom` parses the project files directly in Python (no dbt needed at parse time), and `automatic` falls back through these. `dbt_ls_cache` (via an Airflow Variable) was added to blunt the repeated-parse cost of `dbt_ls`.

`ExecutionMode.local` runs dbt in the Airflow worker's own Python process/venv — fastest, but couples dbt's dependency tree to Airflow's. `virtualenv` builds an isolated venv (optionally persisted) to sidestep that conflict. `kubernetes`/`docker`/cloud-run modes launch each model as a separate pod/container, trading latency for isolation and horizontal scale.

Under the hood every dbt node becomes an operator (`DbtRunLocalOperator`, `DbtTestKubernetesOperator`, etc.), and tests are attached to their model so a failing test fails immediately after the model that produced it, rather than at the end of a monolithic run.

## Production Notes

**DAG-parse cost is the classic footgun.** With `LoadMode.dbt_ls`, Airflow invokes dbt every time the scheduler re-parses the file — which for a large project, multiplied across the parse loop, can dominate scheduler CPU and slow the whole environment. The production-grade pattern is to build `manifest.json` in CI and ship it with the image, then use `LoadMode.dbt_manifest`. This removes dbt from the parse path entirely at the cost of a manifest that can drift from the code.

**Version coupling is real.** Cosmos sits between Airflow's provider ecosystem and dbt's fast-moving releases. A dbt adapter upgrade, a dbt-core minor bump, or an Airflow version change can each require a Cosmos bump; `ExecutionMode.local` makes this worse because dbt and Airflow must resolve a single shared dependency set. Teams that hit adapter/Airflow conflicts usually move to `virtualenv` or `kubernetes` execution specifically to decouple the two dependency trees.

**Connection-to-profile mapping is per-adapter.** Each warehouse has its own `*ProfileMapping` class, and not every dbt profile field is exposed — advanced or newer adapter options sometimes need `profile_args` overrides or a partially hand-managed profile. Verify the mapping covers your auth method (e.g. Snowflake key-pair, BigQuery workload identity) before committing.

**Rendered task volume.** A project with thousands of models produces thousands of Airflow tasks. This is the point of Cosmos, but it stresses the scheduler, the UI grid view, and any per-task overhead. `RenderConfig.select`/`exclude` to split large projects into multiple DAGs is the standard mitigation.

**Telemetry.** Cosmos emits usage telemetry (Scarf) by default; it is documented and can be disabled by operators who need it off.

## When to Use / When Not

**Use when:**
- Airflow is already your orchestrator and you want dbt models as first-class, independently retriable/alertable tasks.
- You want warehouse credentials to come from Airflow connections/secrets rather than a checked-in `profiles.yml`.
- You need dbt to slot between upstream ingestion and downstream tasks in one DAG, with data-aware scheduling.

**Avoid when:**
- You have no Airflow and don't want to run it — dbt Cloud or a plain cron/`dbt build` is far less machinery.
- Your dbt project is tiny and a single `BashOperator` running `dbt build` is genuinely enough.
- You cannot tolerate the operational surface of keeping Cosmos, Airflow, dbt-core, and adapters mutually compatible.

## Alternatives

- dbt-labs/dbt-core — run dbt directly via a `BashOperator`/`KubernetesPodOperator`; choose this when you don't need model-level tasks and want the least moving parts.
- dbt Cloud (dbt-labs, hosted) — use when you want dbt's own managed scheduler/IDE and don't want to operate Airflow.
- apache/airflow (native `KubernetesPodOperator`) — hand-wire dbt runs yourself when you want full control and are willing to write the plumbing Cosmos generates.
- gocardless/airflow-dbt — older community operator that shells out to dbt at the command level; simpler, but no per-model task graph.
- dagster-io/dagster (with dagster-dbt) — pick when you'd rather adopt an asset-based orchestrator built around dbt integration from the start, instead of bolting dbt onto Airflow.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2022-12 | Repository created under Astronomer[^2]. |
| 1.0 | 2023-06 | First stable line; config-object API (`DbtDag`/`DbtTaskGroup`). |
| 1.3 | 2024 | Manifest load mode, caching, and selector improvements. |
| 1.5 | 2024 | Expanded execution modes and profile mappings. |
| 1.x | 2026-07 | Active development; dbt Fusion support alongside dbt Core[^1]. |

Version dates beyond the repo creation are approximate; consult `CHANGELOG.rst` for exact release timing[^3].

## References

[^1]: astronomer-cosmos README and project description — "Run your dbt Core or dbt Fusion projects as Apache Airflow DAGs and Task Groups." https://github.com/astronomer/astronomer-cosmos
[^2]: GitHub API repository metadata (created 2022-12-13), retrieved 2026-07-17.
[^3]: Cosmos CHANGELOG. https://github.com/astronomer/astronomer-cosmos/blob/main/CHANGELOG.rst
[^4]: Cosmos documentation — configuration, execution modes, and load modes. https://astronomer.github.io/astronomer-cosmos/

## Tags

python, airflow, dbt, data-engineering, orchestration, elt, workflow, data-pipeline, apache-airflow, provider-package
