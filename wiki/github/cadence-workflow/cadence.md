# cadence-workflow/cadence

> Durable, event-sourced orchestration engine for long-running business logic — the codebase Temporal was forked from.

[GitHub repo](https://github.com/cadence-workflow/cadence) ·
[Official website](https://cadenceworkflow.io) ·
[License: Apache-2.0](https://github.com/cadence-workflow/cadence/blob/master/LICENSE)

## Overview

Cadence is a workflow orchestration engine originally built at Uber and open-sourced in 2017[^1]. It solves one specific problem: running stateful, long-lived business processes (order fulfillment, provisioning pipelines, human-in-the-loop approvals, sagas) that must survive process crashes, deploys, and machine failures without losing their place. Workflow logic is written as ordinary Go or Java code; the engine records every state transition as an event history and reconstructs in-memory state by deterministically replaying that history — the "durable execution" model.

The single most important fact about this repo is its lineage. Cadence's original authors, Maxim Fateev and Samar Abbas, left Uber and forked Cadence in 2019 to found Temporal[^2]. Temporal is the better-funded, faster-moving successor with a larger community and more SDKs; Cadence continues to be maintained by Uber's platform team and, as of the repo's move out of the `uber/` org into the community `cadence-workflow/` org, a broader open-source contributor base. Anyone evaluating Cadence in 2026 is implicitly choosing it over Temporal, so the comparison is unavoidable and is covered under Alternatives.

At ~9.4k stars with recent commit activity, Cadence is production-proven (it runs a large share of Uber's internal async workflows) but is no longer the default choice for greenfield adopters of the durable-execution pattern. It is chosen mostly by teams already invested in it or with a specific reason to avoid Temporal's licensing/hosting model.

## Getting Started

The backend is a multi-service system plus a database. The fastest local path is Docker Compose:

```bash
git clone https://github.com/cadence-workflow/cadence.git
cd cadence
docker compose -f docker/docker-compose.yml up
# Web UI at http://localhost:8088, frontend gRPC/TChannel on :7833/:7933
```

Workflows themselves live in your own worker process using an SDK. A minimal Go workflow:

```go
func SampleWorkflow(ctx workflow.Context, name string) error {
    ao := workflow.ActivityOptions{
        ScheduleToStartTimeout: time.Minute,
        StartToCloseTimeout:    time.Minute,
    }
    ctx = workflow.WithActivityOptions(ctx, ao)

    var result string
    // Activities run at-least-once and may be retried; the workflow body
    // must stay deterministic across replays.
    err := workflow.ExecuteActivity(ctx, GreetActivity, name).Get(ctx, &result)
    if err != nil {
        return err
    }
    workflow.GetLogger(ctx).Info("greeting done", zap.String("result", result))
    return nil
}
```

The server and CLI install via Homebrew (`brew install cadence-workflow`, which also bundles `cadence-sql-tool` and `cadence-cassandra-tool`) or the `ubercadence/cli` Docker image. Sample recipes are maintained in the separate `cadence-samples` (Go) and `cadence-java-samples` repos.

## Architecture / How It Works

The engine is decomposed into four independently scalable roles, typically run as separate deployments:

- **Frontend** — stateless API gateway. Handles all client RPC (start workflow, signal, query, poll for tasks) and routes to the correct history shard.
- **History** — the core stateful service. Owns workflow mutable state and event histories, sharded by workflow ID across a fixed number of shards. Each shard is owned by exactly one history host at a time (leased via the membership ring).
- **Matching** — hosts task lists (queues). Activity and decision tasks are dispatched to workers that long-poll a named task list. Supports "sticky" task lists so a worker keeps a workflow's cached state.
- **Worker** — internal system worker for background jobs (archival, replication, scanner/fixer for data integrity), distinct from your application workers.

Durability comes from event sourcing. Every meaningful transition — activity scheduled, activity completed, timer fired, signal received — is appended to an immutable history in the persistence layer. When a worker picks up a decision (workflow) task, it replays the history through your workflow code to rebuild state, then asks your code what to do next. This is why **workflow code must be deterministic**: non-deterministic constructs (wall-clock time, random, map iteration order, direct I/O) break replay and are the number-one source of production incidents.

Persistence is pluggable: Cassandra, MySQL, or PostgreSQL for the core store[^3]. Visibility (listing/searching workflows) has two tiers — "basic" backed by the primary DB, and "advanced" backed by Elasticsearch or OpenSearch fed through Kafka, which enables list queries by custom search attributes. Cross-region replication (XDC) is built in for multi-datacenter failover.

## Production Notes

- **Shard count is fixed at cluster creation.** The number of history shards (commonly 512, 1024, 4096, or more) is chosen up front and effectively immutable — resharding is a major migration, not a config change. Under-provisioning caps throughput; wildly over-provisioning wastes resources. This is the single most consequential day-one decision.
- **Cassandra is the most battle-tested backend.** The MySQL/PostgreSQL adapters exist and work, but the highest-scale deployments (including Uber's) run Cassandra, and it receives the most operational attention. Expect SQL backends to be viable for moderate scale but less exercised at extremes.
- **Determinism drift on deploy.** Changing workflow code in a way that alters the sequence of decisions can make in-flight workflows fail to replay. Cadence provides versioning APIs (`workflow.GetVersion`) to gate changes, but forgetting to use them is a classic outage. Long-running workflows can pin you to old code paths for months.
- **Advanced visibility adds real operational surface.** Elasticsearch/OpenSearch plus Kafka is another stateful system to run, size, and keep in sync; index mapping changes for new search attributes require care. Many teams run basic visibility until they genuinely need search.
- **History size limits.** Workflows that loop indefinitely accumulate unbounded event history and hit size/transaction limits; the idiomatic fix is `ContinueAsNew` to start a fresh history. This is a footgun for naive "run forever" designs.
- **Schema upgrades are manual and ordered.** Database schema changes ship as versioned migrations applied with the bundled schema tools; the README specifically warns to remove old Elasticsearch schema before upgrading via Homebrew or index updates silently fail.

## When to Use / When Not

**Use when:**
- You need durable, fault-tolerant execution of long-running or multi-step processes and want to write them as code, not as static DAGs.
- You already run Cadence and it works — there is little reason to migrate away for its own sake.
- You want a fully open-source (Apache-2.0) durable-execution engine with no vendor cloud dependency and no source-available licensing concerns.

**Avoid when:**
- You're starting greenfield with no prior investment: Temporal is the more actively developed successor with more SDKs, docs, and community momentum, and the programming model is nearly identical.
- Your problem is scheduled data pipelines / ETL DAGs — Airflow or Argo fit that shape more directly.
- You lack the operational appetite to run a multi-service stateful system plus Cassandra (and optionally Kafka + Elasticsearch). A managed queue or a simpler job runner may suffice.

## Alternatives

- temporalio/temporal — the fork by Cadence's original authors; choose it for new projects wanting the larger ecosystem and managed-cloud option.
- apache/airflow — use when you need scheduled, DAG-defined data pipelines rather than code-as-workflow durable execution.
- argoproj/argo-workflows — use when your steps are containers and you want Kubernetes-native orchestration.
- indeedeng/iwf — a higher-level DSL/abstraction layer running on top of Cadence or Temporal; use to simplify the raw SDK programming model.
- conductor-oss/conductor — use when you prefer JSON/DSL-defined microservice orchestration over writing workflow code.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016 | Development begins at Uber; authors previously built AWS Simple Workflow (SWF)[^1]. |
| Open source | 2017-02 | Repository published under the `uber/` org[^1]. |
| Fork | 2019 | Original authors leave Uber and fork Cadence to found Temporal[^2]. |
| Org move | — | Repository relocated from `uber/cadence` to `cadence-workflow/cadence` (community org). |
| v1.x | ongoing | Actively maintained; releases tagged on GitHub, CLI distributed via Homebrew and Docker. |

## References

[^1]: Cadence README and project site — "open-source platform since 2017 for building and running scalable, fault-tolerant, and long-running workflows." https://cadenceworkflow.io
[^2]: Temporal documentation and history — Temporal was created by Cadence's original authors (Maxim Fateev, Samar Abbas) after leaving Uber. https://docs.temporal.io/temporal
[^3]: Cadence persistence documentation — Cassandra, MySQL, and PostgreSQL are supported core stores; Elasticsearch/OpenSearch power advanced visibility. https://cadenceworkflow.io/docs/operation-guide/

## Tags

go, java, workflow-orchestration, durable-execution, distributed-systems, event-sourcing, orchestration-engine, uber, microservices, saga, cadence
