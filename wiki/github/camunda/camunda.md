# camunda/camunda

> The Camunda 8 orchestration cluster — a distributed, event-sourced BPMN/DMN process engine (Zeebe) plus its operations and task-management UIs.

[GitHub repo](https://github.com/camunda/camunda) ·
[Official website](https://camunda.com/platform/) ·
License: Camunda License 1.0 (source-available) + Apache-2.0 for clients/protocol[^1]

## Overview

This repository is the monorepo for the server-side components of Camunda 8: the
Zeebe process engine, Operate, Tasklist, Identity, and Optimize[^2]. It is the
successor lineage to `camunda/zeebe` (originally `zeebe-io/zeebe`), which was
folded into this consolidated `camunda/camunda` repo as Camunda unified its
distribution around a single "orchestration cluster." The GitHub description is
"Process Orchestration Framework," but in practice this is a full workflow
platform, not a library.

The defining decision — and the thing that separates Camunda 8 from Camunda 7 —
is that the engine is not an embeddable JAR backed by a relational database. It
is a horizontally scalable, partitioned, event-sourced distributed system.
Processes are modeled visually in BPMN 2.0, decisions in DMN, and application
code talks to the engine as a **client** (over gRPC or REST) rather than calling
an in-process API[^3]. This buys throughput and fault tolerance at the cost of
operational weight: you run a cluster, not a dependency.

The second thing to understand before adopting is licensing. The core engine and
UIs are released under the **Camunda License 1.0**, a source-available license
that permits non-production use but reserves production use for holders of a
commercial agreement; only the Java client, Spring Boot starter, protocol, and
BPMN model API are Apache-2.0[^1]. GitHub reports no SPDX license for this repo
because the primary license is non-OSI. Treat "open source" claims about Camunda
8 with that split in mind.

## Getting Started

The fastest local path is the published Docker image or Docker Compose stack; the
engine also needs Elasticsearch or OpenSearch as its secondary datastore for
Operate/Tasklist/Optimize.

```bash
# Single-broker local cluster (engine + gateway) via the official image
docker run --name zeebe -p 26500:26500 -p 8080:8080 camunda/camunda:latest
```

```java
// Java client — deploy a process and start an instance over gRPC/REST
try (var client = CamundaClient.newClientBuilder()
        .grpcAddress(URI.create("http://localhost:26500"))
        .usePlaintext()
        .build()) {

    client.newDeployResourceCommand()
          .addResourceFromClasspath("order-process.bpmn")
          .send().join();

    client.newCreateInstanceCommand()
          .bpmnProcessId("order-process")
          .latestVersion()
          .variables(Map.of("orderId", "A-1001"))
          .send().join();
}
```

A **job worker** polls the engine for activated jobs of a given type, executes
business logic, and reports completion — the engine never calls your code
directly, which is what makes workers independently scalable and language-agnostic.

## Architecture / How It Works

Zeebe is the heart of the system and the reason the architecture looks unlike a
classic BPM engine:

- **Event sourcing on a replicated log.** Every state change is an event appended
  to a partition's log. Engine state (token positions, variables, timers) is
  materialized into an embedded RocksDB store derived from that log, so state can
  be rebuilt by replay. There is no relational schema for process state[^3].
- **Partitions and Raft.** Throughput scales by adding partitions; each partition
  is a Raft group replicated across brokers for fault tolerance. A process
  instance lives entirely on one partition. Choosing partition count and
  replication factor is an up-front, hard-to-change decision.
- **Gateway + clients.** A stateless gateway fronts the brokers and exposes a gRPC
  API (and, in newer releases, a REST API). Clients and job workers connect to
  the gateway, not to brokers directly.
- **Exporters and CQRS.** The engine does not serve rich queries. Exporters stream
  records off the log into Elasticsearch/OpenSearch; Operate, Tasklist, and
  Optimize read from that secondary store. This is a CQRS split: the write side
  (Zeebe) is fast and append-only, the read side (the UIs) is an eventually
  consistent projection[^2].

The consequence worth internalizing: **what you see in Operate lags the engine.**
Operate, Tasklist, and analytics are views over the exported stream, not live
queries against engine state. Most "why can't I see my instance yet" confusion
traces back to this.

## Production Notes

- **Elasticsearch/OpenSearch is a hard dependency for the UIs.** Zeebe itself
  needs no relational DB, but Operate/Tasklist/Optimize do not function without a
  healthy secondary store. In practice you are operating both a Zeebe cluster and
  an Elasticsearch cluster, plus the exporter pipeline between them.
- **Backpressure is a feature, not a failure.** Under load Zeebe rejects commands
  with `RESOURCE_EXHAUSTED` to protect the log and RocksDB. Clients and workers
  must implement retry/backoff; treating backpressure as an error rather than a
  signal is a common early mistake.
- **Partition count is effectively immutable.** You cannot repartition a live
  cluster arbitrarily; under-provisioning partitions to save resources early is a
  frequent regret at scale.
- **Disk and RocksDB pressure.** State lives on local disk per partition. Long-lived
  process instances, large variable payloads, and slow exporters (which hold the
  log from being compacted) all drive disk growth. Exporter lag stalling log
  compaction is a classic production incident.
- **Rolling upgrades within supported version skew.** Broker upgrades are rolling,
  but only across supported version jumps; skipping minors or mixing incompatible
  broker/gateway versions is unsupported.
- **Licensing gates production.** Running the Camunda-licensed components in
  production requires a commercial agreement[^1]. This is an architecture-shaping
  constraint, not a footnote — budget for it before you build.

## When to Use / When Not

**Use when:**
- You need high-throughput, horizontally scalable, fault-tolerant orchestration
  across services, and can operate a stateful cluster to get it.
- Business analysts model processes in BPMN and you want that model to be the
  executable artifact, with Operate/Tasklist for ops and human tasks.
- You want language-agnostic workers (gRPC/REST clients in Java, Go, Node, Python,
  etc.) rather than in-process engine calls.

**Avoid when:**
- You want an embeddable engine that ships inside your app on a relational DB —
  that is Camunda 7 (`camunda/camunda-bpm-platform`) or Flowable, not this.
- You cannot operate Elasticsearch/OpenSearch and a distributed log, or the
  operational surface outweighs the workflow complexity you actually have.
- Your team prefers workflows-as-code over visual BPMN, or the source-available
  production licensing is a non-starter.

## Alternatives

- camunda/camunda-bpm-platform — Camunda 7, Apache-2.0, embeddable on a relational DB. Use when you want a library inside your app, not a cluster.
- flowable/flowable-engine — Apache-2.0 embeddable BPMN/DMN/CMMN engine, Activiti-lineage. Use when you want open-source, in-process orchestration.
- temporalio/temporal — durable execution as code, no BPMN. Use when engineers own the workflow and visual modeling is unwanted.
- conductor-oss/conductor — JSON-DSL orchestration from the Netflix lineage. Use when you want microservice orchestration without BPMN tooling.
- apache/airflow — DAG scheduler for data pipelines. Use for batch/data workflows, not stateful long-running business processes.

## History

| Version | Date | Notes |
|---------|------|-------|
| Zeebe 0.x | 2017–2020 | Distributed engine incubated as `zeebe-io/zeebe`; pre-1.0 APIs churned heavily. |
| Zeebe 1.0 | 2021-05 | First stable Zeebe; API and gRPC protocol stabilized. |
| Camunda 8.0 | 2022-04 | Camunda 8 GA — Zeebe engine + Operate/Tasklist/Optimize as SaaS and self-managed[^2]. |
| 8.x | 2022–2025 | Roughly twice-yearly minors; added REST API alongside gRPC, OpenSearch support, and component consolidation. |
| Repo consolidation | 2024 | `camunda/zeebe` folded into the unified `camunda/camunda` orchestration-cluster monorepo. |

## References

[^1]: Camunda 8 licensing — Camunda License 1.0 for engine/UIs, Apache-2.0 for Java client, Spring Boot starter, protocol, and BPMN model API. https://github.com/camunda/camunda/blob/main/licenses/CAMUNDA-LICENSE-1.0.txt
[^2]: Camunda 8 component overview (Zeebe, Operate, Tasklist, Identity, Optimize). https://docs.camunda.io/docs/components/
[^3]: Zeebe technical concepts — partitions, Raft, event sourcing, gRPC gateway. https://docs.camunda.io/docs/components/zeebe/technical-concepts/architecture/

## Tags

java, bpmn, dmn, workflow-engine, process-orchestration, event-sourcing, grpc, microservices, distributed-systems, camunda, zeebe, source-available
