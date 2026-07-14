# apache/flink

> A distributed stateful stream processing engine — event-time streaming with exactly-once state, batch as a special case.

[GitHub repo](https://github.com/apache/flink) ·
[Official website](https://flink.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/flink/blob/master/LICENSE)

## Overview

Apache Flink is a distributed engine for stateful computations over unbounded and bounded data streams. It originated from the Stratosphere research project at TU Berlin and became a top-level Apache project in 2014[^1]. Its defining bet, made early and consistently, is that streaming is the primitive and batch is a bounded special case — the opposite of Spark's batch-first lineage. The core value proposition is large, consistent, application-managed state with exactly-once guarantees and true event-time semantics, not just high-throughput record shuffling.

Flink is used where correctness under out-of-order and late data matters and where the pipeline holds meaningful state: fraud detection, real-time ETL and CDC, feature computation for ML, metrics/alerting, and streaming SQL over Kafka. Heavy production users include Alibaba (which acquired the Ververica team behind Flink), Uber, Netflix, and Pinterest. The APIs are layered: low-level `ProcessFunction`, the mid-level DataStream API, and the declarative Table API / Flink SQL that most new pipelines are written in.

The central tension is operational. Flink's model is genuinely more expressive than micro-batch alternatives, but a Flink job is a long-lived stateful distributed system you now operate — checkpoints, state backends, savepoint-based upgrades, and backpressure are your concern, not incidental details. The learning curve is in the operations, not the API.

## Getting Started

Local cluster from a binary distribution:

```bash
# download + unpack a release from https://flink.apache.org/downloads/
./bin/start-cluster.sh          # JobManager + one TaskManager, UI on :8081
./bin/flink run examples/streaming/WordCount.jar
./bin/stop-cluster.sh
```

Maven coordinates for a DataStream job (Java 11/17/21):

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-java</artifactId>
  <version>2.0.0</version>
</dependency>
```

A minimal event-time windowed count:

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStreamSource<String> text = env.socketTextStream("localhost", 9999);

text.flatMap((String line, Collector<String> out) ->
        Arrays.stream(line.split("\\s")).forEach(out::collect))
    .returns(String.class)
    .map(word -> new WordWithCount(word, 1))
    .returns(WordWithCount.class)
    .keyBy(w -> w.word)
    .window(TumblingProcessingTimeWindows.of(Duration.ofSeconds(5)))
    .sum("count")
    .print();

env.execute("word-count");
```

## Architecture / How It Works

A running job is a **JobManager** (coordinator: scheduling, checkpoint orchestration, failure recovery) and one or more **TaskManagers** (workers, each offering task slots). The user program is compiled into a dataflow graph of operators, then parallelized: each operator becomes N subtasks distributed across slots. Records flow through the graph; keyed streams are hash-partitioned by key so all state for a key lives on one subtask.

**State and checkpointing are the core.** Operators hold keyed state (per-key values, lists, maps) and operator state. Flink snapshots this state consistently using *asynchronous barrier snapshotting*, a variant of the Chandy–Lamport algorithm[^2]: barriers are injected into the stream at the sources and flow with the records; when an operator has received barrier *n* on all inputs it snapshots its state. This produces a globally consistent checkpoint without stopping the stream. On failure the whole job rewinds to the last completed checkpoint and replays from the recorded source offsets.

**State backends** decide where that state lives. The hashmap backend keeps state on the JVM heap (fast, bounded by memory). The **RocksDB backend** keeps state in an embedded LSM store on local disk, enabling state far larger than RAM at the cost of serialization and disk I/O on every access. Flink 2.0 (2025) added a *disaggregated state* backend that stores state on remote storage (e.g. object stores) to decouple state size from local disk[^3].

**Exactly-once** is end-to-end only when sinks cooperate. Internal state consistency comes from checkpoints; delivery to external systems requires either idempotent writes or the two-phase-commit sink protocol, where the sink pre-commits on checkpoint and commits on checkpoint completion (Kafka transactions, transactional file sinks). Without a transactional sink the guarantee degrades to at-least-once.

**Time and watermarks.** Event time is driven by watermarks — assertions that no event older than time *t* will arrive. Windows and timers fire on watermark progress. This is what makes correct out-of-order processing possible, and also the most common source of "my window never fires" confusion.

**Savepoints** are user-triggered, self-contained checkpoints used for upgrades: stop a job to a savepoint, deploy new code, restore. State schema and operator UIDs must stay compatible across the restore, which is what makes application evolution a real design constraint rather than a redeploy.

## Production Notes

**Checkpointing is the tuning surface.** Checkpoint interval, timeout, and alignment mode trade recovery time against runtime overhead. Under backpressure, aligned checkpoints stall behind buffered data; **unaligned checkpoints** (Flink 1.11+) let barriers overtake in-flight buffers at the cost of larger snapshots. Slow or failing checkpoints are the number-one operational symptom and usually trace to state size, sink stalls, or backpressure.

**RocksDB is where large jobs live and where they hurt.** State access becomes serialization + disk I/O, so hot per-record state lookups can dominate CPU. Practical levers: local SSD (never network disk) for the RocksDB directory, incremental checkpoints to avoid re-uploading full state, managed memory sizing, and timers-in-RocksDB vs heap. Watch for state that grows unbounded because a `MapState` key or window is never cleaned up — set state TTL.

**Upgrades are savepoint-gated.** You cannot freely rename operators or change state types. Assign stable `uid()` to every stateful operator from day one; without them, restoring a savepoint after a topology change silently drops state. Cross-major upgrades (1.x → 2.0) removed long-deprecated APIs including the legacy Scala DataStream/DataSet APIs and `SourceFunction`/`SinkFunction`, so migrations are non-trivial[^3].

**Backpressure propagates.** A slow sink slows the entire pipeline via credit-based flow control; the UI's backpressure and busy metrics are the first place to look. There is no magic buffer that absorbs a permanently-slow downstream.

**Deployment modes matter.** Application mode (one cluster per job, `main()` runs on the JobManager) is the recommended production shape; session mode (shared cluster, many jobs) risks noisy-neighbor and classloader interference; per-job mode is deprecated. The adaptive scheduler and reactive mode support rescaling, but state redistribution on rescale is real work, not instant.

**Connectors are externalized.** Kafka, JDBC, Elasticsearch, and most others live in separate `flink-connector-*` repositories with their own release cadence and version matrices; a connector compatible with your Flink version is a real dependency-management task, not a given.

## When to Use / When Not

**Use when:**
- You need large, consistent, application-managed state with exactly-once guarantees.
- Event-time correctness under out-of-order and late data is a hard requirement.
- The workload is a long-lived low-latency pipeline (CDC, fraud, alerting, streaming joins/aggregations).
- You want one engine spanning streaming and bounded/batch execution with shared SQL.

**Avoid when:**
- The job is stateless or nearly so — a lighter tool (Kafka Streams, a consumer loop) removes the operational tax.
- You only run periodic batch analytics — Spark or a warehouse is a better fit.
- The team can't own a stateful distributed system: checkpoint tuning, savepoint upgrades, and state backends are ongoing operational work.
- You want SQL-over-streams without running a cluster — a streaming database may be simpler.

## Alternatives

- apache/spark — use Structured Streaming instead when your workload is batch-first and micro-batch latency (seconds) is acceptable.
- apache/kafka — use Kafka Streams instead when the app is a per-key transformation that can run as a colocated library rather than a managed cluster.
- apache/beam — use Beam instead when you need a portable pipeline API across runners (Flink can itself be the Beam runner).
- risingwavelabs/risingwave — use instead when you want streaming as a Postgres-compatible SQL database and don't want to operate a Flink cluster.
- apache/storm — largely superseded; consider only for legacy at-least-once topologies without Flink's state model.

## History

| Version | Date | Notes |
|---------|------|-------|
| incubation | 2014 | Stratosphere donated to Apache; becomes top-level project[^1]. |
| 1.0 | 2016-03 | First stable release; API stability guarantees begin. |
| 1.4 / 1.5 | 2017–2018 | Two-phase-commit sinks; new deployment/resource model. |
| 1.9 | 2019-08 | Blink (Alibaba) SQL/planner merge begins. |
| 1.11 | 2020-07 | Unaligned checkpoints; new Table/SQL improvements; CDC momentum. |
| 1.13 | 2021-05 | Reactive mode / adaptive scheduler for rescaling. |
| 1.15 | 2022-05 | Scala-free core artifacts; connectors externalized. |
| 1.20 | 2024-08 | Final 1.x line, positioned as an LTS bridge to 2.0. |
| 2.0 | 2025-03 | First major in ~9 years: disaggregated state, deprecated-API removal[^3]. |

## References

[^1]: Apache Flink — project history and Stratosphere origin. https://flink.apache.org/what-is-flink/history/
[^2]: Carbone et al., "Lightweight Asynchronous Snapshots for Distributed Dataflows" (2015). https://arxiv.org/abs/1506.08603
[^3]: Apache Flink blog, "Apache Flink 2.0.0: A new Era of Real-Time Data Processing." https://flink.apache.org/2025/03/24/apache-flink-2.0.0-a-new-era-of-real-time-data-processing/

## Tags

java, stream-processing, big-data, distributed-systems, event-time, exactly-once, stateful, streaming-sql, dataflow, batch-processing, apache
