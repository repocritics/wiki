# apache/nifi

> Visual, flow-based data routing and transformation — a directed graph of processors moving FlowFiles between systems, with full provenance.

[GitHub repo](https://github.com/apache/nifi) ·
[Official website](https://nifi.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/nifi/blob/main/LICENSE)

## Overview

Apache NiFi is a dataflow system built on flow-based programming: you assemble a directed graph of **processors** on a browser canvas, wire them with queued **connections**, and units of data called **FlowFiles** move through the graph while NiFi records the lineage of every one[^1]. It originated inside the U.S. NSA as a project called "Niagarafiles," was donated to the Apache Software Foundation in 2014, and became a top-level project in 2015[^2]. It is written in Java and targets integration/movement problems — ingesting from and delivering to files, databases, message brokers, HTTP endpoints, cloud object stores, and hundreds of other systems via bundled connectors.

The defining tradeoff is **operator-friendly visual dataflow versus code-first pipeline engineering**. NiFi's strengths are that non-programmers can build and modify running pipelines through the UI with no deploy step, that back-pressure and guaranteed delivery are built into every connection, and that provenance gives searchable "where did this byte come from" lineage out of the box. The cost is that the flow lives as canvas state rather than reviewable code, per-FlowFile overhead makes it a poor fit for very high-throughput per-record stream computation, and a busy NiFi is a stateful, disk-and-heap-hungry service that demands real operational care.

NiFi ships alongside two sibling projects in the same repository: **NiFi Registry** (flow version control) and **MiNiFi** (a lightweight edge agent that runs a subset of the same components).

## Getting Started

Requires Java 21; Python 3.10+ is optional and only needed for native Python processors[^3].

```shell
# Download a binary release from nifi.apache.org/download, unpack, then:
cd nifi-*/
./bin/nifi.sh start                      # starts on https://localhost:8443/nifi
grep Generated logs/nifi-app*log         # find the auto-generated single-user credentials
```

Set your own credentials instead of the random ones:

```shell
./bin/nifi.sh set-single-user-credentials <username> <password>
```

Build from source with the bundled Maven wrapper:

```shell
./mvnw install -T1C -am -pl :nifi-assembly   # framework + assembly only, parallel build
```

There is no meaningful "hello world" in code — the first flow is built by dragging a `GenerateFlowFile` processor and a `LogAttribute` processor onto the canvas and drawing a connection between them.

## Architecture / How It Works

The unit of data is a **FlowFile**: a pointer to a content blob plus a map of string **attributes** (metadata). Processors act on FlowFiles and route them to named **relationships** (`success`, `failure`, etc.); a **connection** is a queue between two processors and is where back-pressure and prioritization live.

State is held in three repositories, and understanding them is the key to operating NiFi:

- **FlowFile Repository** — a write-ahead log of the live state and attributes of every in-flight FlowFile. Attributes live in JVM heap while a FlowFile is queued.
- **Content Repository** — the actual content bytes, stored immutably. Transformations are copy-on-write; the original content is retained until claim references drop, which is what makes replay and provenance possible.
- **Provenance Repository** — an append-only, indexed event log (CREATE, CLONE, ROUTE, DROP, SEND, RECEIVE...) that reconstructs the lineage graph and enables content replay.

**Back-pressure** is per-connection: when a queue exceeds its object-count threshold (default 10,000) or data-size threshold, the upstream processor stops being scheduled. Large queues **swap** FlowFile metadata to disk to bound heap. Connections can prioritize (FIFO, oldest-first, by attribute) and, since 1.8, load-balance across cluster nodes.

**Clustering** is zero-master: every node runs the identical flow and processes its own share of the data. ZooKeeper elects a **cluster coordinator** and a **primary node** (the node that runs "primary-only" source processors to avoid duplicate ingestion). Data is *not* automatically rebalanced across nodes — a connection's load-balancing strategy must be set explicitly, and adding/removing nodes does not redistribute already-queued data.

Extensions are packaged as **NAR** files (NiFi Archive), which give each bundle an isolated classloader so conflicting dependency versions can coexist. Flow-to-flow transfer between NiFi instances uses the **Site-to-Site** protocol.

## Production Notes

**Disk is the first thing to fill.** The content and provenance repositories grow with throughput and retention settings, not with the number of active FlowFiles. Provenance in particular can consume large amounts of disk and I/O; tune retention (`nifi.provenance.repository.max.storage.time` / `.size`) and put the three repositories on separate physical volumes to avoid I/O contention. A full disk stalls the whole flow.

**Heap pressure comes from FlowFile *count* and attribute size, not content size.** Millions of small FlowFiles, or processors that stuff large values into attributes, blow up heap because attributes are held in memory for queued FlowFiles. Prefer record-oriented processors (`*Record` with readers/writers) to process many logical records inside one FlowFile rather than one-FlowFile-per-record.

**It is not a stream-processing engine.** Per-FlowFile scheduling overhead makes NiFi excellent at routing/mediation/enrichment but a poor substitute for Kafka Streams or Flink when you need windowed stateful computation over millions of events per second. NiFi commonly sits *in front of* those systems, not in place of them.

**The flow is stateful runtime configuration, not code.** Changes take effect live with no deploy, which is a double-edged sword: it is easy to break production by editing the canvas, and reviewing/diffing changes requires NiFi Registry (flow versioning) plus discipline. Treat Registry as mandatory for any team environment.

**Security defaults and history.** Recent versions are secure-by-default: HTTPS on 8443 with a self-signed cert and a generated single-user login. Production needs a real CA cert plus OIDC/SAML and policy-based authorization. NiFi has a meaningful CVE history (including remote-code-execution issues in specific processors and expression-language sinks); keep it patched and restrict which processors untrusted users can configure.

**Upgrades: 1.x → 2.x is a migration, not a bump.** NiFi 2.0 moved to Java 21, replaced `flow.xml.gz` with `flow.json.gz`, removed a large set of long-deprecated components and the variable registry/templates, and introduced the native Python processor API and a rebuilt UI. Audit your flow for removed components before attempting the jump[^4].

## When to Use / When Not

**Use when:**
- You need to move and transform data between many heterogeneous systems and want visual, live-editable pipelines.
- Guaranteed delivery, back-pressure, and end-to-end data provenance/lineage are hard requirements.
- Operators or analysts (not just engineers) need to build and adjust flows.
- Ingestion/mediation at the edge or in front of a message bus or lake.

**Avoid when:**
- You need high-throughput stateful stream computation (windowing, joins, aggregations) — use Flink or Kafka Streams.
- You want pipelines as reviewable, testable, version-controlled code first — use Airflow, Dagster, or Camel.
- Your workload is scheduled batch orchestration of external jobs rather than continuous dataflow.
- You cannot afford a stateful, disk-heavy JVM service to operate and tune.

## Alternatives

- apache/airflow — use instead when you want code-defined DAGs orchestrating scheduled batch jobs rather than continuous FlowFile routing.
- apache/kafka — use instead when you need a durable, replayable event log and pub/sub backbone; NiFi often feeds it rather than replacing it.
- apache/flink — use instead when the core need is stateful, low-latency stream computation over high event volumes.
- apache/camel — use instead when you want code-first enterprise integration patterns and version control over a visual canvas.
- airbytehq/airbyte — use instead when the problem is prebuilt ELT connectors syncing sources into a warehouse, not general dataflow.

## History

| Version | Date | Notes |
|---------|------|-------|
| (incubation) | 2014-11 | Donated to ASF as "Niagarafiles"; enters Apache Incubator[^2]. |
| TLP | 2015-07 | Graduates to Apache top-level project[^2]. |
| 1.0.0 | 2016 | Zero-master clustering, multi-tenant authorization. |
| 1.8.0 | 2018 | Load-balanced connections; NiFi Registry maturing for flow versioning. |
| 1.x | 2016–2024 | Long-lived line on Java 8/11; broad connector growth. |
| 2.0.0 | 2024–2025 | Java 21, native Python processors, rebuilt UI, `flow.json.gz`, deprecated components removed[^4]. |

## References

[^1]: Apache NiFi overview and core concepts. https://nifi.apache.org/documentation/
[^2]: Apache NiFi project history / ASF graduation announcement. https://nifi.apache.org/
[^3]: Apache NiFi README — platform requirements (Java 21, optional Python 3.10+). https://github.com/apache/nifi/blob/main/README.md
[^4]: Apache NiFi 2.0 migration guidance. https://cwiki.apache.org/confluence/display/NIFI/Migration+Guidance

## Tags

java, dataflow, etl, data-integration, flow-based-programming, streaming, data-pipeline, provenance, apache, ingestion, connectors
