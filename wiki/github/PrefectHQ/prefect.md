# PrefectHQ/prefect

> A Python workflow orchestration framework that turns ordinary functions into scheduled, retried, observable pipelines — no static DAG required.

[GitHub repo](https://github.com/PrefectHQ/prefect) ·
[Official website](https://prefect.io) ·
[License: Apache-2.0](https://github.com/PrefectHQ/prefect/blob/main/LICENSE)

## Overview

Prefect is a workflow orchestration framework for building data pipelines in Python[^1]. You decorate plain functions with `@flow` and `@task`, and the runtime adds scheduling, retries, caching, concurrency limits, state tracking, and a monitoring UI. The pitch is incremental adoption: an existing script becomes an orchestrated workflow by adding two decorators, without rewriting it into a foreign DAG abstraction first.

The defining architectural decision — and the thing that separates Prefect from Airflow — is that **there is no static DAG**. Flows are just Python; the execution graph is discovered at runtime as tasks are called. Loops, conditionals, and dynamically-sized fan-out all work because the "graph" is whatever the code does when it runs. This buys enormous flexibility (native control flow, parametrized runs, data-dependent branching) at the cost of losing a graph you can inspect before execution — you cannot render the full topology of a run until it has actually run.

The other structural fact worth understanding up front is the **open-source / Cloud split**. The Prefect library, the orchestration server, and the UI are all Apache-2.0 and can be fully self-hosted[^2]. Prefect Cloud is the commercial hosted control plane that adds the multi-tenant features an OSS server does not ship: RBAC, SSO, audit logs, service accounts, and managed infrastructure[^3]. Everything you write against the OSS SDK runs unchanged against Cloud — the split is in the control plane, not the workflow code.

## Getting Started

Prefect requires Python 3.10+.

```bash
pip install -U prefect
# or: uv add prefect
```

```python
from prefect import flow, task
import httpx


@task(retries=3, log_prints=True)
def get_stars(repo: str) -> int:
    count = httpx.get(f"https://api.github.com/repos/{repo}").json()["stargazers_count"]
    print(f"{repo}: {count} stars")
    return count


@flow(name="GitHub Stars")
def github_stars(repos: list[str]):
    for repo in repos:              # dynamic fan-out — no DAG declaration
        get_stars(repo)


if __name__ == "__main__":
    github_stars(["PrefectHQ/prefect"])
```

Run a local server and open the UI at `http://localhost:4200`:

```bash
prefect server start
```

To schedule it, turn the flow into a deployment:

```python
github_stars.serve(name="hourly", cron="0 * * * *",
                   parameters={"repos": ["PrefectHQ/prefect"]})
```

## Architecture / How It Works

Prefect has four moving parts that are easy to conflate:

1. **The SDK** (`@flow` / `@task`) — decorators that intercept function calls, submit them to a task runner, and report state transitions to the API. Task runners are pluggable: the default `ThreadPoolTaskRunner` runs tasks concurrently in threads; `prefect-dask` and `prefect-ray` distribute across clusters.
2. **The orchestration API / server** — a FastAPI + SQLAlchemy service backed by SQLite (default) or PostgreSQL. It stores flow/task run state, schedules, logs, and events, and enforces orchestration rules (retries, concurrency limits, cache lookups).
3. **Work pools + workers** — the deployment execution layer. A deployment targets a *work pool* (e.g. a Docker, Kubernetes, or process pool); a *worker* polls that pool and launches each scheduled run in the appropriate infrastructure. In Prefect 3 this replaced the older 2.x "agents" model[^4].
4. **Blocks and results** — *blocks* are typed, serializable configuration objects (credentials, storage backends, infra config) stored in the API and referenced by name. *Results* are task return values persisted to configurable storage so that caching and retries can restore them.

State is the spine of the whole system: every task and flow run moves through a state machine (`Pending → Running → Completed/Failed/Cached/…`), and orchestration logic — retries, caching, cancellation — is implemented as rules on those transitions rather than as graph scheduling. Prefect 3 added **transactions**: groups of tasks with commit/rollback semantics and `on_rollback` hooks, so a failed downstream task can undo an upstream side effect. Prefect 3 also moved the **events and automations** engine (event-driven triggers, e.g. "run flow B when flow A emits event X") into the open-source server; earlier it was largely Cloud-only.

## Production Notes

**SQLite is a toy for real workloads.** The default server backend is SQLite, which serializes writes and falls over under concurrent flow runs. Any multi-worker or high-throughput self-host must point `PREFECT_API_DATABASE_CONNECTION_URL` at PostgreSQL. This is the single most common self-hosting mistake.

**The OSS server historically had no authentication.** For most of Prefect 2.x/3.x's life the self-hosted UI and API shipped with no built-in auth — you were expected to put it behind a reverse proxy, VPN, or network boundary. Basic auth has since been added, but do not assume a fresh `prefect server start` is safe to expose publicly.

**Ephemeral mode is opt-in and packaged separately.** Running against an in-process ephemeral API (no long-running server) requires the `prefect[server]` extra; the lean `prefect-client` package deliberately omits it. Ephemeral mode is convenient for tests and scripts but is not a production runtime.

**Workers are long-lived infrastructure you have to run.** Deployments do not execute themselves — a worker process must be polling the work pool. If no worker is up, scheduled runs pile up in `Scheduled`/`Late` and nothing tells you loudly. Monitoring worker liveness is on you (Cloud has automations for this; OSS needs external alerting).

**Dynamic graphs cost you static analysis.** Because the DAG only exists at runtime, you cannot pre-validate topology, estimate a run's shape, or get Airflow-style "here is the whole pipeline" views before execution. Debugging a large fan-out means reading logs, not a static graph.

**Version migrations are real events.** 1.x → 2.x was a full rewrite with no in-place upgrade. 2.x → 3.x removed agents in favor of workers, changed result/caching internals, and moved to Pydantic v2 — pin your Prefect version and read the migration guide before bumping a major.

## When to Use / When Not

**Use when:**
- Your pipelines are Python and you want orchestration without adopting a separate DAG DSL.
- You need data-dependent control flow: loops, conditionals, dynamic fan-out sized at runtime.
- You want a low-friction path from local script to scheduled deployment, with the option of a managed control plane later.
- Retries, caching, and per-task observability matter more than a static, inspectable graph.

**Avoid when:**
- You need the pipeline topology known and validated before it runs (static DAG, lineage, impact analysis) — Airflow/Dagster fit better.
- Your team isn't Python-centric, or workflows are primarily SQL transformations (dbt is the right layer).
- You want a batteries-included managed scheduler and don't want to operate Postgres + workers yourself — evaluate a hosted product end-to-end first.
- You need durable, long-running, human-in-the-loop workflows with strong execution guarantees — a durable-execution engine like Temporal is a different tool.

## Alternatives

- apache/airflow — use instead when you want a mature, static-DAG scheduler with the largest operator/provider ecosystem and don't need dynamic runtime graphs.
- dagster-io/dagster — use instead when you want asset-centric orchestration with built-in data lineage and typed IO managers.
- kestra-io/kestra — use instead when you prefer declarative YAML workflows and a language-agnostic engine over Python-native code.
- temporalio/temporal — use instead when you need durable execution and long-running stateful workflows rather than data-pipeline scheduling.
- spotify/luigi — use instead for a lightweight, dependency-resolution batch scheduler when you don't need a server or UI.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2018–2021 | Prefect Core: DAG-based, flows registered as static graphs. |
| 1.0 | 2022 | Final stable line of the classic DAG engine. |
| 2.0 | 2022-07 | "Orion" rewrite — dynamic flows/tasks, no static DAG, runtime graph[^1]. |
| 3.0 | 2024-09 | Transactions, events/automations in OSS, workers replace agents, Pydantic v2[^4]. |

## References

[^1]: Prefect documentation. https://docs.prefect.io
[^2]: Prefect self-hosting guide. https://docs.prefect.io/v3/manage/self-host
[^3]: Prefect Cloud vs. open source. https://www.prefect.io/cloud-vs-oss
[^4]: Work pools & workers — Prefect docs. https://docs.prefect.io/v3/deploy/infrastructure-concepts/work-pools

## Tags

python, workflow-orchestration, data-engineering, data-pipelines, scheduler, mlops, dag, observability, etl, apache-2.0
