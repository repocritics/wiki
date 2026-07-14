# kestra-io/kestra

> Event-driven orchestration platform where workflows are declared in YAML and the engine runs on the JVM.

[GitHub repo](https://github.com/kestra-io/kestra) ·
[Official website](https://kestra.io) ·
[License: Apache-2.0](https://github.com/kestra-io/kestra/blob/develop/LICENSE)

## Overview

Kestra is an orchestration and scheduling platform for data, AI, and infrastructure
workflows, built by Kestra Technologies (founded 2019, first open-source releases in
2021)[^1]. Its defining choice is that workflows ("flows") are declared in YAML rather
than written as code in a host language. A flow is a namespace-scoped document listing
`tasks` and `triggers`; the actual work — running Python, shell, SQL, container jobs,
cloud API calls — is delegated to plugins. This puts Kestra in a different lane from the
Python-first orchestrators (Airflow, Dagster, Prefect): the orchestration layer is
language-agnostic and declarative, and your code stays in the runtime it belongs to.

The engine itself is Java (Micronaut) with a Vue.js UI, distributed as a single
container image[^2]. That JVM core is the first tension: teams expecting a `pip install`
Python tool get a server platform with a database, an internal queue, and a worker pool.
The second tension is the open-core boundary. The Apache-2.0 repository here is the full
execution engine, but multi-tenancy, RBAC, SSO, worker groups, and several secret and
governance features live only in the closed-source Enterprise Edition[^3]. Much of the
friction operators hit is discovering which capability is OSS and which is EE.

Kestra is used for data pipelines (ETL/ELT), scheduled batch jobs, event-driven
automation reacting to files/queues/webhooks, and increasingly for stitching together AI
and infrastructure tasks. It competes both with data orchestrators and, on the low-code
automation side, with tools like n8n.

## Getting Started

```bash
docker run --pull=always -it -p 8080:8080 --user=root \
  --name kestra --restart=always \
  -v kestra_data:/app/storage \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp:/tmp \
  kestra/kestra:latest server local
```

The `server local` mode runs every component in one process with an embedded H2
database — fine for evaluation, not for production. Open the UI at `http://localhost:8080`
and create a flow:

```yaml
id: hello_world
namespace: company.team

inputs:
  - id: name
    type: STRING
    defaults: World

tasks:
  - id: say_hello
    type: io.kestra.plugin.core.log.Log
    message: "Hello, {{ inputs.name }}!"

triggers:
  - id: daily
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 9 * * *"
```

Task and input references use Pebble templating (`{{ ... }}`) — the expression language
for interpolating inputs, outputs of prior tasks, execution metadata, and variables.

## Architecture / How It Works

Kestra separates the control plane into distinct components that communicate over an
internal queue and share a repository[^2]:

- **Executor** — consumes execution events, walks the flow graph, decides which task runs
  next. Holds no user code; it dispatches work.
- **Worker** — actually executes task runs (scripts, container jobs, plugin logic).
- **Scheduler** — evaluates triggers (cron schedules, flow/event conditions) and creates
  executions.
- **Webserver** — REST API + UI.
- **Indexer** — present in the Kafka/Elasticsearch topology, copies queue state into the
  search index.

The queue and repository are **pluggable backends**, and this is the most important
architectural fact to internalize:

1. **JDBC backend** (H2, PostgreSQL, MySQL) — the queue and repository are both the
   database. PostgreSQL is the recommended production choice. Simpler to run; scaling is
   bounded by the database.
2. **Kafka + Elasticsearch backend** — the queue is Kafka, the search/repository is
   Elasticsearch. This is the high-availability topology for large deployments and is
   associated with the Enterprise Edition.

A flow is a DAG of tasks, but Kestra also supports sequential/parallel task groups,
subflows, `ForEach`/dynamic tasks, conditional branching, retries, timeouts, and explicit
error handlers. Task **outputs** and **artifacts** are captured and surfaced in the UI.
Plugins are separate JARs dropped onto the classpath (the `kestra/kestra:latest-full`
image bundles the common set); each plugin ships its own versioned task types under a
reverse-DNS namespace like `io.kestra.plugin.core.log.Log`. Internal storage for artifacts
and inter-task files is itself pluggable: local filesystem, S3, GCS, Azure Blob, or MinIO.

## Production Notes

**`server local` is not production.** It uses embedded H2 and runs everything in one
process. Production means an external PostgreSQL (or the Kafka/ES topology) and, ideally,
separately scaled Executor/Worker/Scheduler processes via `server executor`,
`server worker`, etc. Running a single `server standalone` is the common middle ground.

**The OSS/EE line is a recurring surprise.** Multi-tenancy, RBAC, SSO/OIDC, worker
groups, audit logs, and several secret backends are Enterprise-only[^3]. In OSS,
tenant isolation is not available — everything shares one tenant — and secrets are handled
through environment variables and the built-in KV store rather than a full secrets manager.
Teams that architect around per-team isolation in OSS hit a wall.

**Plugin version skew.** Because plugins are independently versioned JARs, the plugin set
baked into your image can drift from the flows that reference it. Pinning the image tag
(not `latest`) and tracking plugin versions matters; a task type or property can change
between plugin releases even when the core engine version is stable.

**JVM footprint and tuning.** The engine is Java, so expect JVM memory characteristics —
heap sizing (`JAVA_OPTS`) matters for busy workers, and idle memory use is higher than a
comparable Python scheduler. Worker concurrency is bounded by CPU and heap, not just a
config number.

**YAML at scale.** Declarative YAML is approachable for a first flow and gets verbose for
large branching pipelines. Pebble expressions add real logic but are string-templated, so
mistakes surface at runtime rather than at edit time despite the UI's live validation.
Subflows and namespace-level files help; treat flows as versioned code in Git from the
start (the Git integration and Terraform provider exist for exactly this).

**Upgrades.** Kestra is pre-1.0 and iterates quickly; the `develop` default branch and
frequent releases mean reading release notes before bumping is non-optional, as flow
properties and plugin APIs do change across minor versions.

## When to Use / When Not

**Use when:**
- You want orchestration decoupled from any single language — polyglot teams running
  Python, shell, SQL, and containers under one scheduler.
- You want declarative, Git-versioned workflows plus a UI editor rather than
  code-as-DAG.
- You need both scheduled and event-driven triggers (files, queues, webhooks) in one tool.
- You're comfortable operating a JVM server with PostgreSQL behind it.

**Avoid when:**
- You want a lightweight, pip-installable, Python-native tool with dynamic
  code-defined DAGs — Airflow/Dagster/Prefect fit that better.
- Your workflows are fundamentally Python programs and YAML would just wrap Python calls.
- You need multi-tenancy, RBAC, or SSO without paying for Enterprise.
- You want durable code-level execution semantics for microservices — Temporal is the
  right shape.

## Alternatives

- apache/airflow — the incumbent; Python DAGs, vast provider ecosystem. Use instead when
  your team is Python-first and wants the largest integration catalog.
- dagster-io/dagster — asset-oriented, data-aware orchestration in Python. Use instead
  when you model pipelines as data assets with lineage and typing.
- PrefectHQ/prefect — Python-native, dynamic flows with minimal ceremony. Use instead when
  you want to decorate existing Python functions into a pipeline.
- temporalio/temporal — durable execution for code-first microservice/workflow logic. Use
  instead when you need long-running, fault-tolerant workflows written as code.
- windmill-labs/windmill — turns scripts into workflows with a developer platform. Use
  instead when you want script-to-app with a lighter footprint.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-08 | Repository created; Kestra Technologies founded[^1]. |
| 0.x (early) | 2021 | First public open-source releases[^1]. |
| 0.x (ongoing) | 2022–2026 | Steady minor-version cadence: plugin ecosystem growth, UI editor, task runners, KV store, Git/Terraform integrations. Enterprise Edition adds multi-tenancy, RBAC, SSO, worker groups[^3]. |

Exact latest version and its date were not verified from a live source at authoring time;
the project remains pre-1.0 and releases frequently. Consult the releases page[^4].

## References

[^1]: Kestra — About / company. https://kestra.io/about-us
[^2]: Kestra docs — Architecture. https://kestra.io/docs/architecture
[^3]: Kestra — Enterprise Edition. https://kestra.io/enterprise
[^4]: Kestra releases. https://github.com/kestra-io/kestra/releases

## Tags

java, jvm, workflow-orchestration, data-orchestration, scheduler, event-driven, yaml, etl, data-engineering, low-code, self-hosted, open-core
