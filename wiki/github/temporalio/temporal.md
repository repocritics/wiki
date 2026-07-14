# temporalio/temporal

> Durable execution platform — application code runs as workflows whose full state survives crashes, restarts, and multi-day waits.

[GitHub repo](https://github.com/temporalio/temporal) ·
[Official website](https://docs.temporal.io) ·
[License: MIT](https://github.com/temporalio/temporal/blob/main/LICENSE)

## Overview

Temporal is the open-source server behind a "durable execution" model: you write ordinary
functions (Workflows) in a general-purpose language, and the platform guarantees they run to
completion exactly as written, even if the process executing them dies mid-way. State is not
checkpointed by you — it is reconstructed by replaying an event history. This lets a workflow
`sleep` for 30 days, survive a deploy, and resume on the exact next line.

Temporal originated as a 2019 fork of Uber's Cadence, built by Cadence's original authors
(Maxim Fateev, Samar Abbas) at Temporal Technologies[^1]. The server reached 1.0 in September
2020[^2]. This repository is the **server** only — the Frontend/History/Matching/Worker
services plus persistence and the `tctl`/`temporal` CLI. Application logic lives in separate SDK
repos (Go, Java, TypeScript, Python, .NET, PHP, Ruby), and Workers running your code connect to
the server over gRPC.

The defining tension is that durability is bought with **determinism**. Workflow code must be
replayable: given the same event history, it must take the same path every time. That constraint
(no direct clocks, randomness, network calls, or non-deterministic map iteration in workflow code)
is the whole conceptual cost of the model, and the source of most production incidents.

## Getting Started

Run a local dev server (in-memory, single binary):

```bash
brew install temporal          # or download the Temporal CLI
temporal server start-dev      # UI at http://localhost:8233, gRPC on :7233
```

A minimal Go Workflow + Activity (application code, using `go.temporal.io/sdk`):

```go
// A Workflow: deterministic, replayable orchestration.
func GreetingWorkflow(ctx workflow.Context, name string) (string, error) {
    ctx = workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        StartToCloseTimeout: time.Minute, // retries are automatic by default
    })
    var greeting string
    err := workflow.ExecuteActivity(ctx, ComposeGreeting, name).Get(ctx, &greeting)
    return greeting, err
}

// An Activity: the side-effecting work (I/O, network, non-determinism lives here).
func ComposeGreeting(ctx context.Context, name string) (string, error) {
    return "Hello " + name, nil
}
```

A Worker process registers these with a Task Queue and long-polls the server for work; the client
starts executions with `client.ExecuteWorkflow(...)`. The server never runs your code — it only
schedules tasks and stores history.

## Architecture / How It Works

Temporal is **event-sourced**. Every workflow has an append-only history (WorkflowTaskScheduled,
ActivityTaskCompleted, TimerFired, …). When a Worker picks up a workflow task, it replays that
history through your code to rebuild in-memory state, then executes forward until the next command.
The server is the durable log and scheduler; the SDK is the replay engine[^3].

The server splits into four independently scalable gRPC services:

- **Frontend** — the API gateway; rate-limits and routes all client/Worker calls.
- **History** — the core. Owns workflow mutable state, the event history, and transfer/timer
  task queues. Sharded (default 512 shards, fixed at cluster creation) for horizontal scale.
- **Matching** — hosts Task Queues; matches pending tasks to polling Workers.
- **Worker** (internal) — runs Temporal's own system workflows (archival, scheduled batch, etc.),
  not your code.

**Persistence** is pluggable: Cassandra, PostgreSQL, or MySQL for the core store. **Visibility**
(the "list/search workflows" query surface) is a second store — basic visibility runs on the SQL
DB, while advanced visibility (SearchAttributes, complex queries) historically needed Elasticsearch,
though modern Postgres/MySQL support advanced visibility natively[^4].

Key primitives: **Activities** (retriable side effects), **Signals** (async input into a running
workflow), **Queries** (synchronous read of workflow state), **Timers**, **Child Workflows**, and
**Continue-As-New** (atomically restart a workflow with fresh history to bound history growth).
Cross-service calls between namespaces/teams go through **Nexus**, a newer operation-based RPC layer.

## Production Notes

**History size is a hard limit, not a soft one.** A single workflow execution caps at ~51,200
events and ~50 MB of history; the server warns at ~10k/10MB. Long-running or high-iteration
workflows (e.g. a per-event loop) must use Continue-As-New to reset history, or they will be
terminated. This is the number-one durable-execution footgun.

**Determinism / versioning.** Any change to workflow code that alters the command sequence will
break replay of in-flight executions ("nondeterminism error"). You cannot just edit and redeploy.
The fixes are `workflow.GetVersion` / patching or Worker Build-ID-based Versioning to gate changes
by a version marker. Reordering statements, changing a loop, or upgrading a library that changes
map iteration can all silently break replay.

**Operating the cluster is real work.** Sizing Cassandra/Postgres, tuning the 512-shard count
(immutable after creation — pick high), running and upgrading Elasticsearch for advanced
visibility, and monitoring the History service's persistence latency are ongoing burdens. Many
teams that adopt Temporal ultimately move to **Temporal Cloud** (the company's managed offering)
specifically to shed persistence operations.

**Task queue backpressure and poller starvation.** If Workers are undersized, tasks sit in
Matching and latency climbs invisibly; if a "sticky" Worker cache is evicted, the whole history
replays on the next task, spiking CPU. Sticky execution caching and right-sized Worker pools matter.

**Upgrades** are frequent (minor releases roughly monthly) and generally require staying within a
supported version skew; schema migrations ship with server versions. Read the release notes — some
minors change defaults for retention, visibility, or task processing.

## When to Use / When Not

**Use when:**
- Long-running, stateful, multi-step processes must survive failures (order fulfillment, payments,
  provisioning, human-in-the-loop, saga/compensation flows).
- You want retries, timeouts, and state persistence as language-level constructs, not hand-rolled
  queue + DB + cron plumbing.
- Exactly-once effect semantics and full auditability of every step matter.

**Avoid when:**
- The work is a simple fire-and-forget job or a data pipeline — a queue (SQS/RabbitMQ) or an
  Airflow/Argo DAG is lighter and cheaper.
- You cannot accept the operational footprint of a sharded stateful cluster (and Temporal Cloud is
  off the table for cost/compliance).
- Your logic is inherently non-deterministic and can't be cleanly split into Activities.
- Sub-millisecond latency per step is required — Temporal adds persistence round-trips per task.

## Alternatives

- uber/cadence — the predecessor Temporal forked from; similar model, smaller ecosystem and slower
  cadence now. Use only if already invested in it.
- conductor-oss/conductor — Netflix-origin orchestration with JSON/DSL-defined workflows; use when
  you prefer declarative workflow definitions over workflow-as-code.
- apache/airflow — DAG scheduler for batch/data pipelines; use for scheduled ETL and analytics, not
  low-latency transactional workflows.
- argoproj/argo-workflows — Kubernetes-native, each step is a container; use when steps are already
  containerized k8s jobs.
- restatedev/restate — newer durable-execution engine, single binary, lighter to operate; use when
  you want durable execution without Temporal's cluster footprint.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017 | Uber open-sources Cadence, the direct predecessor. |
| — | 2019-10 | temporalio/temporal repo created; fork from Cadence begins[^1]. |
| v0.10.0 | 2020-02-05 | Early public releases under the Temporal name. |
| v1.0.0 | 2020-09-30 | Server GA; stable API and persistence schema[^2]. |
| v1.13.0 | 2021-10-29 | Continued sharding/visibility maturation. |
| v1.16.0 | 2022-04-12 | Advanced visibility on SQL stores progresses[^4]. |
| v1.31.2 | 2026-07-08 | Recent stable release (Nexus, Build-ID versioning mature). |

## References

[^1]: Temporal — "About / origins as a fork of Uber Cadence." https://temporal.io/
[^2]: temporalio/temporal release v1.0.0 — 2020-09-30. https://github.com/temporalio/temporal/releases/tag/v1.0.0
[^3]: Temporal server architecture docs (in-repo). https://github.com/temporalio/temporal/blob/main/docs/architecture/README.md
[^4]: Temporal docs — "Visibility" (basic vs advanced, Elasticsearch vs SQL). https://docs.temporal.io/self-hosted-guide/visibility

## Tags

go, durable-execution, workflow-engine, orchestration, distributed-systems, event-sourcing, microservices, saga, self-hosted, backend-infrastructure
