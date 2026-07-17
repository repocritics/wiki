# robinhood/faust

> Kafka Streams, ported to Python as an asyncio library — now deprecated in favor of a community fork.

[GitHub repo](https://github.com/robinhood/faust) ·
[Documentation](https://faust.readthedocs.io/) ·
[License: BSD-3-Clause](https://github.com/robinhood/faust/blob/master/LICENSE)

## Overview

Faust is a stream processing library that ports the ideas of Kafka Streams to
Python, built on `asyncio` and static type annotations[^1]. It was created at
Robinhood by Ask Solem (author of Celery) and open-sourced in 2017[^2]. The
pitch: express stream topologies as ordinary `async def` functions ("agents")
over Kafka topics, with no DSL and no JVM — you keep NumPy, Pandas, Django, and
the rest of the Python ecosystem in-process. Robinhood used it to process
billions of events per day.

The single most important fact about this repository is that **it is
deprecated**. The README carries a deprecation notice directing users to the
community fork `faust-streaming/faust`, which is where bug fixes, Python 3.8+
support, and dependency updates now land[^3]. The original repo was effectively
abandoned after Solem left Robinhood; the last upstream release was 1.10.4. The
GitHub repo still receives occasional housekeeping commits, but no functional
maintenance. New projects should not depend on `robinhood/faust` — this page
documents it because a large body of existing code, blog posts, and Stack
Overflow answers still reference the original coordinates.

The defining tension is that Faust is genuinely pleasant to write against —
stream processing as plain Python async is a real ergonomic win over the
Kafka Streams / Flink / Spark model — while being operationally heavy and, in
its original form, unmaintained. The good ideas survived; they just live under
a different owner now.

## Getting Started

For any new work, install the maintained fork rather than the deprecated
package (the fork publishes to the same import name `faust`):

```bash
pip install faust-streaming
pip install "faust-streaming[rocksdb,uvloop]"   # production table store + fast loop
```

A minimal app defining a model, a Kafka topic, and an agent:

```python
import faust

app = faust.App('myapp', broker='kafka://localhost:9092')

class Order(faust.Record):
    account_id: str
    amount: int

orders_topic = app.topic('orders', value_type=Order)

@app.agent(orders_topic)
async def process(orders):
    async for order in orders:              # infinite async stream
        print(f'Order for {order.account_id}: {order.amount}')

if __name__ == '__main__':
    app.main()
```

Run a worker: `faust -A myapp worker -l info`. A windowed table (counts per URL
over the last hour) looks like a dict backed by RocksDB and a Kafka changelog:

```python
counts = app.Table('click_counts', default=int).tumbling(3600.0)

@app.agent(app.topic('clicks', key_type=str, value_type=int))
async def count(clicks):
    async for url, n in clicks.items():
        counts[url] += n
```

## Architecture / How It Works

Faust sits on top of **aiokafka** (the asyncio Kafka client) and **mode**, an
async service/actor framework also written by Solem[^1]. The programming
surface has three primitives:

- **Agents** — `async def` functions decorated with `@app.agent(topic)`. Each
  agent is a long-lived coroutine consuming a topic; concurrency is set per
  agent, and agents can call `.ask()` / `.send()` to talk to each other,
  giving a lightweight actor model.
- **Models** (`faust.Record`) — typed serialization schemas using variable
  annotations. JSON by default; codecs cover others.
- **Tables** — named distributed key/value stores that behave like Python
  dicts. State lives locally in an embedded **RocksDB** store (C++), and every
  mutation is published to a Kafka **changelog** topic used as a write-ahead
  log. Standby replicas consume the changelog so a failed partition can be
  recovered on another worker.

Partitioning is the core scaling mechanism, inherited directly from Kafka
Streams: the topic key determines the partition, and Kafka's consumer-group
rebalancing assigns partitions to workers. State for a key always lands on the
worker owning that key's partition, so table lookups are local. Windowing
(tumbling, hopping, sliding) is built on top of tables with time-bucketed keys
and expiry.

Because each worker is a single `asyncio` event loop, Faust scales by running
**many worker processes**, not threads — one per core, coordinated through
Kafka's group protocol. This is the mental model to internalize: Faust is not a
cluster scheduler like Flink; it is a library where Kafka itself is the
coordinator, and every operational property (ordering, parallelism, recovery)
is a consequence of Kafka partition semantics.

## Production Notes

- **Rebalancing is the dominant operational pain.** Every worker
  start/stop/crash triggers a Kafka consumer-group rebalance that pauses
  processing across the group. Under frequent deploys or flapping workers this
  causes visible latency spikes. Table-heavy apps make it worse: a partition
  moving to a new worker may require replaying the changelog into a fresh
  RocksDB store before processing resumes.
- **RocksDB recovery can be slow.** Cold recovery of a large table from the
  changelog is bounded by changelog size, not table size, unless log compaction
  is configured on the changelog topic. Enable compaction and standby replicas
  (`table_standby_replicas`) to keep failover fast.
- **Joins require co-partitioning.** Like Kafka Streams, joining two streams
  needs both topics partitioned by the same key with the same partition count.
  Mismatches silently route data to the wrong worker.
- **RocksDB file-descriptor limits.** A common local failure is
  "max open files exceeded" from RocksDB; raise the OS `ulimit`[^1].
- **`at-least-once` by default.** Exactly-once semantics are limited; design
  agents to be idempotent and expect reprocessing after failures.
- **Dependency rot on the original repo.** `robinhood/faust` pins old
  `aiokafka`, `mode`, and `python-rocksdb` versions that conflict with modern
  Python. Installing on Python 3.10+ frequently fails; this is the practical
  reason the fork exists and why migrating to `faust-streaming` is usually
  mandatory rather than optional[^3].

## When to Use / When Not

**Use when:**
- You want Kafka-native stream processing but the team is Python, not JVM, and
  you want to reuse Python data/ML libraries in the processing path.
- Your topology is expressible as agents + tables and you already run Kafka.
- You are maintaining an existing Faust codebase (migrate to the fork).

**Avoid when:**
- You are starting fresh and evaluating the original `robinhood/faust` — use
  the maintained fork, or a newer library, instead.
- You need exactly-once end-to-end guarantees or complex event-time joins at
  scale — Flink is the more complete engine.
- You are not on Kafka. Faust is Kafka-only; there is no pluggable broker.
- You want a supported product with an SLA — this is a library whose original
  authors have moved on.

## Alternatives

- faust-streaming/faust — the actively maintained community fork; use this instead of `robinhood/faust` for essentially all cases.
- quixio/quix-streams — modern pure-Python Kafka stream processing with a pandas-like API; use when starting fresh and you want active commercial backing.
- bytewax/bytewax — Python dataflow with a Rust (Timely Dataflow) core; use when you want higher throughput and non-Kafka sources.
- apache/flink (PyFlink) — heavyweight JVM engine with real exactly-once and event-time; use when correctness guarantees and scale outweigh operational cost.
- confluentinc/confluent-kafka-python — lower-level client; use when you want to build processing logic yourself without a framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-sourced | 2017-03 | Released by Robinhood; ports Kafka Streams to Python asyncio[^2]. |
| 1.0 | 2018 | First stable line; agents, tables, RocksDB store, windowing. |
| 1.10.4 | 2020 | Final release from Robinhood before the project stalled. |
| fork | 2020 | Community fork `faust-streaming/faust` created to continue maintenance[^3]. |
| deprecation notice | — | README updated to redirect users to the fork. |
| last commit | 2024-07 | Occasional housekeeping only; no functional maintenance. |

## References

[^1]: Faust documentation. https://faust.readthedocs.io/
[^2]: Robinhood Engineering, "Faust: stream processing for Python." https://github.com/robinhood/faust
[^3]: faust-streaming/faust — maintained community fork. https://github.com/faust-streaming/faust

## Tags

python, asyncio, kafka, stream-processing, distributed-systems, kafka-streams, event-processing, rocksdb, deprecated, data-pipeline
