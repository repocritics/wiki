# quixio/quix-streams

> A pure-Python stream processing library for Apache Kafka, using a pandas-like StreamingDataFrame API instead of a JVM engine.

[GitHub repo](https://github.com/quixio/quix-streams) ·
[Official website](https://docs.quix.io) ·
[License: Apache-2.0](https://github.com/quixio/quix-streams/blob/main/LICENSE)

## Overview

Quix Streams is a client-side Python library for building stateful stream processing applications on top of Apache Kafka[^1]. It is not a cluster or a server: it is a library you `pip install` into an ordinary Python process, which consumes from Kafka topics, transforms messages, and produces results back to Kafka. It occupies the same conceptual niche as Kafka Streams (JVM) or Faust (Python), but with a declarative, pandas-flavored `StreamingDataFrame` API as the primary interface. It is developed by Quix, a company that also sells a managed platform (Quix Cloud), though the library is fully usable standalone.

The single most important fact about this project is that **the code you find in old tutorials is probably for a different library.** Versions through the 0.x line (up to 0.5.7, 2023) were a thin Python wrapper around a C#/.NET interop core — a fundamentally different architecture and API[^2]. The 2.x line (alpha in late 2023, stable in early 2024) was a ground-up rewrite to "pure Python, no wrappers around Java," introducing the `StreamingDataFrame` model that defines the project today[^3]. Anything referencing `StreamData`, `TimeseriesData`, or the C#-derived classes predates the rewrite and does not apply to current versions.

The defining tradeoff is Python-versus-JVM. You gain native debugging, direct use of the Python data/ML ecosystem (NumPy, pandas, scikit-learn, PyTorch) inside your processing logic, and no cross-language boundary. You pay for it in raw throughput: per-message processing runs in CPython, single-threaded per process, and is bound by the GIL and Python overhead rather than the JVM's JIT. This is a reasonable trade for many operational-analytics and ML-inference pipelines; it is a poor one for CPU-bound processing at Kafka Streams / Flink scale.

## Getting Started

```shell
python -m pip install quixstreams
# or: conda install -c conda-forge quixio::quixstreams
```

Requires Python 3.9+ and a reachable Apache Kafka 0.10+ broker.

```python
from quixstreams import Application

# Read Celsius temperatures from one topic, convert to Fahrenheit,
# and emit alerts above a threshold to another topic.
app = Application(broker_address="localhost:9092")

temperature_topic = app.topic("temperature-celsius", value_deserializer="json")
alerts_topic = app.topic("temperature-alerts", value_serializer="json")

sdf = app.dataframe(topic=temperature_topic)
sdf = sdf.apply(lambda v: {"temperature_F": (v["temperature"] * 9 / 5) + 32})
sdf = sdf[sdf["temperature_F"] > 150]          # filter on a derived column
sdf = sdf.to_topic(alerts_topic)               # produce results

app.run()                                      # blocking consume/process/produce loop
```

## Architecture / How It Works

There are two core objects. `Application` owns the Kafka lifecycle — consumer/producer setup, deserialization, checkpointing, commit, and teardown. `StreamingDataFrame` (SDF) is a **declarative, lazily-built pipeline**: calls like `.apply()`, `.update()`, `.filter()`, and `.to_topic()` register operations rather than executing immediately. When `app.run()` starts, each incoming message is pushed through the registered operations one at a time. Despite the name and the `df["col"]` indexing syntax, an SDF is not pandas — there is no batch DataFrame in memory; it is a per-message streaming abstraction that borrows pandas ergonomics.

Scaling and fault tolerance are inherited directly from Kafka's consumer-group model. You do not configure parallelism in the library; you run more copies of the same process in the same consumer group, and Kafka assigns partitions across them. Maximum useful parallelism equals the topic's partition count. A single process handles its assigned partitions in one thread.

**State** is the substantive part. Stateful operations (windowing, aggregations, joins, group-by) persist their state in an embedded RocksDB store on local disk, partitioned to match Kafka partitions[^4]. State changes are mirrored to Kafka **changelog topics**, so if a process dies or a partition is reassigned during a rebalance, the new owner rebuilds state by replaying the changelog — the same recovery mechanism Kafka Streams uses. This gives durability without an external database, at the cost of local disk and rebuild time.

Other internals worth knowing:

- **Windowing** — tumbling, hopping, and sliding windows, all state-backed; window state must expire or it grows without bound.
- **Serializers** — JSON, Avro, and Protobuf, with Confluent Schema Registry integration.
- **Sources & Sinks** — a connector API for pulling non-Kafka data in and pushing processed data out (databases, object stores, etc.), so Kafka can sit in the middle of an ETL path.
- **Exactly-once** — optional, implemented via Kafka transactions on the produce+commit path.
- **Joins & GroupBy** — stream-to-stream joins and repartitioning operators, both leaning on the same state/changelog machinery.

## Production Notes

**State is local, and that shapes your deployment.** RocksDB state lives on the container's disk. If you run in Kubernetes without a persistent volume, every restart discards local state and forces a full changelog replay before processing resumes. For large state this recovery is not instantaneous — it can be minutes. Either attach persistent volumes (and accept sticky scheduling) or budget for changelog-rebuild time on every deploy and every rebalance.

**Rebalances are the main operational hazard.** Adding/removing instances, deploys, or a slow consumer triggers a Kafka group rebalance; partitions (and their state) move between processes. During this window processing pauses and reassigned partitions rebuild state. Frequent rebalances — from aggressive autoscaling or unstable pods — can keep a pipeline perpetually recovering. Tune `session.timeout.ms` / `max.poll.interval.ms` and avoid thrashing the instance count.

**Throughput is Python-bound.** One process = one processing thread. You scale out to partition count, not beyond, and each partition's work is CPython-speed. CPU-heavy per-message logic (deserialization of large payloads, ML inference) is the usual bottleneck. If you need more parallelism than partitions allow, you must repartition the topic upstream. This is a genuine ceiling versus JVM engines.

**Exactly-once has a cost.** Kafka transactions add latency and reduce throughput versus at-least-once, and require broker support and correct `transactional.id` handling across instances. Default to at-least-once with idempotent downstream writes unless you truly need exactly-once semantics.

**The rewrite is a documentation minefield.** Because 0.x and 2.x+ are effectively different libraries, third-party blog posts, StackOverflow answers, and older Quix samples may be silently incompatible. Pin versions explicitly and treat any material predating the `StreamingDataFrame` API as obsolete. Within the 3.x line the API has been comparatively stable, but the ecosystem's collective memory still contains the old world.

**Vendor gravity is mild but present.** The library runs anywhere Python does and does not require Quix Cloud. The official docs, however, consistently route toward the managed platform for deployment and monitoring. Self-hosting is fully supported; you will just be assembling your own CI/CD, observability, and scaling around it.

## When to Use / When Not

**Use when:**
- Your team is Python-first and wants stream processing without introducing a JVM stack.
- Your processing logic needs the Python data/ML ecosystem inline (feature engineering, model inference on a stream).
- You are already on Kafka and want stateful operations (windows, joins, aggregations) without standing up Flink.
- You want a library, not a cluster — deployable as an ordinary container/process.

**Avoid when:**
- Throughput per partition is CPU-bound and must match JVM Kafka Streams or Flink.
- You are not on Kafka (or a Kafka-API-compatible broker like Redpanda) — the library is Kafka-native, not a general dataflow engine.
- You need a mature, distributed, checkpoint-to-object-store engine for very large windowed state — Flink is the heavier but more battle-tested choice.
- You prefer to express stream processing in SQL rather than Python.

## Alternatives

- faust-streaming/faust — the older Python streaming library (agent/actor model, community-maintained after Robinhood); use it if you specifically want Faust's agent style, though Quix Streams is the more actively developed pure-Python option.
- bytewax/bytewax — Rust-core Python dataflow; use when you want a dataflow model that is not tied exclusively to Kafka.
- apache/flink — use when you need a mature distributed engine for large-scale, heavily stateful/windowed workloads (PyFlink gives a Python surface over the JVM engine).
- apache/kafka — its Kafka Streams library; use when you are on the JVM and want the reference stream-processing implementation with maximum throughput.
- confluentinc/ksql — ksqlDB; use when you would rather define stream processing declaratively in SQL, server-side, than in application code.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.5.x | 2023 | Last of the legacy line — a Python wrapper over a C#/.NET interop core[^2]. |
| 2.0 (alpha) | 2023-11 | First public alpha of the pure-Python rewrite; `StreamingDataFrame` introduced[^3]. |
| 2.x | early 2024 | Rewrite stabilizes; RocksDB state, windowing, serializers. |
| 3.0.0 | 2024-10-10 | Major version; expanded Sources/Sinks, joins, group-by[^5]. |
| 3.24.0 | 2026-06-12 | Recent release in an actively maintained, roughly monthly cadence[^5]. |

## References

[^1]: Quix Streams README and documentation. https://quix.io/docs/quix-streams/introduction.html
[^2]: Legacy 0.x releases (through v0.5.7, 2023) predate the rewrite; the GitHub release history shows the 0.5.x line ending in 2023 before the 2.x alphas. https://github.com/quixio/quix-streams/releases
[^3]: First pure-Python alpha tag `v2.0alpha2`, published 2023-11-08, introduced the `StreamingDataFrame` API. https://github.com/quixio/quix-streams/releases
[^4]: Stateful processing / state store documentation (RocksDB, changelog recovery). https://quix.io/docs/quix-streams/advanced/stateful-processing.html
[^5]: GitHub releases — v3.0.0 (2024-10-10) through v3.24.0 (2026-06-12). https://github.com/quixio/quix-streams/releases

## Tags

python, kafka, stream-processing, streaming-dataframe, event-driven, data-engineering, real-time, stateful-processing, rocksdb, etl, machine-learning
