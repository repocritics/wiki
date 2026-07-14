# apache/airflow

> A platform to author, schedule, and monitor batch workflows defined as Python code.

[GitHub repo](https://github.com/apache/airflow) ·
[Official website](https://airflow.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/airflow/blob/main/LICENSE)

## Overview

Airflow is the de facto open-source scheduler for batch data pipelines. Workflows are written as Python — a DAG (directed acyclic graph) file declares tasks and their dependencies, and a long-running scheduler decides when each task runs based on a schedule and upstream state[^1]. It was created at Airbnb in 2014 (Maxime Beauchemin), entered the Apache incubator in 2016, and became a top-level ASF project in 2019. As of 2026 it has ~46k stars and remains the most widely deployed workflow orchestrator, with a large provider ecosystem covering most clouds and databases.

The defining design opinion is that **workflows are mostly static, slowly changing, and idempotent**[^2]. Airflow schedules and monitors work; it is not built to pass large data between tasks (the XCom mechanism is for small metadata, backed by the metadata database) and it is not a streaming engine. Heavy lifting is expected to be delegated to external systems (Spark, dbt, warehouses, Kubernetes jobs) with Airflow orchestrating the sequencing and retries.

The defining tension is that Airflow is "a bit of both a library and an application"[^3]. It ships as the `apache-airflow` PyPI package but expects to be run as stateful infrastructure — a scheduler, a metadata database, workers, and a web UI. Getting a correct install is famously fiddly because dependencies are kept deliberately open, and the project ships pinned "constraints" files to make installs reproducible.

## Getting Started

```bash
# Install a pinned, known-good build (do not rely on bare `pip install apache-airflow`)
pip install 'apache-airflow==3.3.0' \
  --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-3.3.0/constraints-3.10.txt"

airflow standalone   # runs scheduler + API/UI + a SQLite metadata DB for local dev only
```

```python
# dags/etl.py — TaskFlow API (Airflow 2.0+)
from airflow.decorators import dag, task
import pendulum

@dag(schedule="@daily", start_date=pendulum.datetime(2026, 1, 1), catchup=False)
def etl():
    @task
    def extract() -> list[int]:
        return [1, 2, 3]

    @task
    def transform(rows: list[int]) -> int:
        return sum(rows)

    @task
    def load(total: int) -> None:
        print(f"loaded {total}")

    load(transform(extract()))

etl()
```

Drop the file in the configured `dags/` folder; the scheduler parses it and the run appears in the Grid view.

## Architecture / How It Works

Airflow is several cooperating processes around a shared **metadata database** (Postgres or MySQL in production; SQLite is dev-only[^2]):

- **Scheduler** — parses DAG files on a loop, evaluates dependencies and schedules, and enqueues task instances. Historically a single point of failure; Airflow 2.0 made it horizontally scalable and highly available (run multiple active schedulers)[^4].
- **Executor** — decides *where* tasks run. This is the most consequential deployment choice: `LocalExecutor` (subprocesses on one host), `CeleryExecutor` (a Celery worker pool + Redis/RabbitMQ broker), and `KubernetesExecutor` (one pod per task) are the common ones, plus Celery/Kubernetes hybrids.
- **Workers** — actually execute task code.
- **API server / UI** — a web app for triggering, inspecting, and debugging runs (Grid, Graph, and Dag views).

A DAG file is ordinary Python that runs at parse time. Tasks are **operators** (`PythonOperator`, `BashOperator`, and hundreds of provider operators) or TaskFlow-decorated functions. Data does not flow through Python return values across process boundaries by default — cross-task values move through **XCom**, serialized into the metadata DB, which is why passing large payloads is an anti-pattern.

Provider packages are versioned and released independently of core (`apache-airflow-providers-google`, `-amazon`, etc.), each following its own SemVer[^5]. This is why "which Airflow" is really "core plus a specific set of provider versions."

**Airflow 3.0 (2025)** was the first major architectural change since 2.0: a Task Execution API and Task SDK so workers no longer talk directly to the metadata database (a long-standing security and multi-tenancy concern), DAG versioning, a rewritten React UI, and the renaming/expansion of Datasets into **Assets** for data-aware scheduling[^6].

## Production Notes

**Top-level DAG-file code is the classic footgun.** Everything at module scope runs every time the scheduler parses the file — often every few seconds, for every DAG. A DB query, API call, or heavy import at the top level multiplies across the whole DAG set and stalls the scheduler. Keep expensive work inside task bodies; the parse loop is not the place for it.

**Executor choice dominates operational cost.** `KubernetesExecutor` gives isolation and elastic scaling but pays pod-startup latency per task (bad for many short tasks). `CeleryExecutor` reuses warm workers (low latency) but you now operate a broker and a worker fleet. Picking wrong is the most common source of either cost blowups or scheduling lag.

**The metadata database is the real bottleneck.** XCom, task history, and rendered templates all land there; high task volumes make it the scaling limit. Retention/cleanup of old runs (`airflow db clean`) is mandatory maintenance, not optional.

**Installs and upgrades are painful by design.** Because dependencies are open, `pip install apache-airflow` without the matching constraints file frequently produces a broken environment[^3]. Only `pip` is officially supported — Poetry and pip-tools are not. Provider versions drift against core, so upgrades are a coordination exercise, not a one-line bump.

**Scheduling semantics changed under people.** The historical `execution_date` / interval model confused nearly everyone; timetables and the data-interval model replaced the mental model over the 2.x line, and Airflow 3 continues to reshape scheduling. Migrations across majors (1.10 → 2.0, 2.x → 3.0) are non-trivial and warrant a staged plan with a DB backup.

**Not for low-latency or sub-minute work.** Scheduler loop latency, parse intervals, and executor startup mean Airflow measures in seconds-to-minutes. It is a batch orchestrator; event-driven or streaming needs a different tool.

## When to Use / When Not

**Use when:**
- You have scheduled, batch, mostly-static pipelines (ETL/ELT, ML training/retraining, reporting) that orchestrate external systems.
- You want pipelines as Python code with retries, backfills, SLAs, and a real UI for monitoring.
- You need broad, maintained connectors to clouds, warehouses, and databases via the provider ecosystem.

**Avoid when:**
- You need streaming, sub-second, or truly event-driven execution.
- Your tasks pass large data payloads between steps (Airflow wants that in external storage).
- You want a lightweight, zero-ops scheduler — running Airflow well is a standing infrastructure commitment.
- You need per-run dynamic graph shapes beyond what dynamic task mapping supports; a code-first data framework may fit better.

## Alternatives

- dagster-io/dagster — asset/software-defined-assets model with strong local dev and typing; use instead when you want data assets and testability as first-class, not task DAGs.
- PrefectHQ/prefect — Python-native flows with dynamic runtime graphs; use instead when pipelines are highly dynamic and you dislike the parse-time DAG model.
- kestra-io/kestra — declarative YAML orchestration on the JVM; use instead when you want language-agnostic, config-driven pipelines over Python code.
- apache/dolphinscheduler — visual, UI-first DAG scheduler; use instead when non-engineers build workflows through a GUI.
- temporalio/temporal — durable execution for long-running stateful workflows; use instead when you need code-level durability and retries for application workflows rather than batch data scheduling.

## History

| Version | Date | Notes |
|---------|------|-------|
| incubator | 2016-03 | Enters Apache incubator; created at Airbnb 2014[^1]. |
| 1.10.0 | 2018-08 | Long-lived 1.x line; RBAC UI, Kubernetes executor. |
| (TLP) | 2019-01 | Graduates to Apache top-level project. |
| 2.0.0 | 2020-12 | HA scheduler, TaskFlow API, stable REST API, provider packages[^4]. |
| 2.3.0 | 2022-05 | Dynamic task mapping. |
| 2.4.0 | 2022-09 | Datasets — data-aware scheduling. |
| 3.0.0 | 2025-04 | Task Execution API/Task SDK, DAG versioning, React UI, Assets[^6]. |
| 3.3.0 | 2026 | Current stable line (2.11.2 is the deprecated 2.x line)[^2]. |

## References

[^1]: Apache Airflow documentation — project overview. https://airflow.apache.org/docs/apache-airflow/stable/
[^2]: apache/airflow README — Project Focus, Requirements, and version table. https://github.com/apache/airflow
[^3]: apache/airflow README — "Installing from PyPI" (library-vs-application dependency stance and constraints files). https://github.com/apache/airflow#installing-from-pypi
[^4]: Apache Airflow blog, "Apache Airflow 2.0 is here!" — 2020-12-17. https://airflow.apache.org/blog/airflow-two-point-oh-is-here/
[^5]: Airflow provider packages index. https://airflow.apache.org/docs/apache-airflow-providers/index.html
[^6]: Apache Airflow 3.0 release announcement. https://airflow.apache.org/blog/airflow-three-point-oh-is-here/

## Tags

python, workflow-orchestration, data-engineering, etl, scheduler, dag, data-pipelines, batch-processing, mlops, apache
