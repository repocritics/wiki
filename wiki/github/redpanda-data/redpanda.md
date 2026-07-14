# redpanda-data/redpanda

> A Kafka-API-compatible streaming data platform written in C++, with no ZooKeeper and no JVM — a single self-contained broker binary.

[GitHub repo](https://github.com/redpanda-data/redpanda) ·
[Official website](https://redpanda.com) ·
License: BSL 1.1 + Redpanda Community License (source-available, not OSI open-source)[^1]

## Overview

Redpanda is a streaming data platform that speaks the Apache Kafka wire protocol but replaces the entire Kafka/ZooKeeper JVM stack with a single C++ binary[^2]. It was built by Redpanda Data (formerly Vectorized), first published in 2020, with the explicit goal of being a drop-in Kafka replacement that is simpler to operate and lower-latency, particularly at the tail. Existing Kafka clients, `librdkafka`-based tools, Kafka Connect, and most of the Kafka ecosystem talk to it unmodified.

The defining engineering bet is the **Seastar** framework (the same thread-per-core, shared-nothing runtime that powers ScyllaDB). Each CPU core owns a shard of partitions, its own memory, and its own network queues; cores communicate by explicit message passing rather than shared locks. This, combined with direct/DMA disk I/O and a built-in Raft consensus layer instead of Kafka's ISR-plus-ZooKeeper model, is where the "10x faster / lower p99" marketing claims originate. Those numbers are vendor benchmarks — the architecture genuinely removes JVM GC pauses and a coordination service, but real-world gains depend heavily on hardware and workload.

The honest tension around Redpanda is licensing, not engineering. The core is **not** open source in the OSI sense: it ships under the Business Source License 1.1 (each version converts to Apache 2.0 four years after release) with certain enterprise features gated behind the proprietary Redpanda Community License[^1]. Teams that adopt it for "open Kafka without the ops" should read the license before assuming Apache-Kafka-equivalent freedoms.

## Getting Started

```bash
# macOS (Docker required) — single-node dev cluster
brew install redpanda-data/tap/redpanda
rpk container start                 # spins up a local broker + console

# Debian/Ubuntu production package
curl -1sLf 'https://linux.pkg.redpanda.com/setup-redpanda.deb.sh' | sudo -E bash
sudo apt-get install redpanda
```

`rpk` is the unified CLI (cluster admin, topic ops, producing/consuming):

```bash
rpk topic create orders -p 6 -r 3          # 6 partitions, replication factor 3
echo '{"id":1,"item":"panda"}' | rpk topic produce orders
rpk topic consume orders --num 1           # read one record back
```

Any Kafka client works by pointing at the broker (default `localhost:9092`) — no Redpanda-specific SDK is required.

## Architecture / How It Works

- **Thread-per-core (Seastar).** Redpanda pins one reactor thread per core and shards partition leadership across cores. There is effectively no shared mutable state between cores, so there are no cross-core locks on the hot path. The cost: it wants dedicated, isolated CPUs and hugepages, and it is unhappy sharing cores with noisy neighbors.
- **Raft everywhere.** Every partition is a Raft group; replication, leader election, and durability are handled by Raft directly. This removes ZooKeeper (and, unlike Kafka's KRaft, was the design from day one) and gives per-partition consensus rather than a separate ISR mechanism. `acks=all` maps to a Raft majority commit.
- **Single binary, batteries included.** The broker also embeds an HTTP proxy (pandaproxy), a Schema Registry, and an Admin API. There is no separate connect/registry process to run for those functions.
- **Tiered storage.** Log segments can be offloaded to object storage (S3/GCS/Azure) and read back transparently, decoupling retention from local disk. This is an enterprise (RCL) feature.
- **Data Transforms (WASM).** Stateless in-broker transforms run as WebAssembly modules, an alternative to a separate stream-processing tier for simple map/filter work.
- **rpk / Console.** `rpk` is the Go CLI in this repo; the web Console (Redpanda Console, formerly Kowl) lives in a separate repository.

Versioning is calendar-based (`YY.MAJOR.PATCH`, e.g. `25.2.x`), with a handful of feature releases per year and patch releases in between.

## Production Notes

- **CPU pinning is not optional for peak numbers.** The advertised latency depends on Seastar owning full physical cores with `--overprovisioned` disabled. In containers/Kubernetes without CPU pinning and hugepages, you get correctness but not the benchmark tail latency. Redpanda ships a `--overprovisioned` mode specifically for shared/dev environments.
- **Memory model differs from Kafka.** Redpanda manages its own memory (Seastar allocates a large slab up front) rather than relying on the OS page cache the way Kafka does. Sizing is per-core memory, not heap tuning — a genuinely different mental model for Kafka operators.
- **Partition count still matters.** "No ZooKeeper" removes one scaling ceiling, but very high partition counts still cost memory and Raft bookkeeping per shard. It is not unbounded.
- **Licensing gates real features.** Tiered storage, some security/RBAC, audit logging, and continuous data balancing are RCL/enterprise. Building a "free open Kafka" plan on top of these will hit a license or feature wall.
- **Ecosystem edges.** Because it reimplements the protocol, occasional gaps or version-lag against newer Kafka protocol features or specific admin RPCs show up before they are closed. Most clients are fine; exotic broker-side integrations should be validated.
- **Build from source is heavy.** The build uses Bazel/bazelisk and pulls a large C++ toolchain and dependency tree; most users should consume prebuilt packages or containers rather than compile.

## When to Use / When Not

**Use when:**
- You want Kafka semantics and clients but do not want to run ZooKeeper/KRaft + JVM broker fleets.
- Tail latency and predictable p99 matter (financial, adtech, real-time pipelines).
- You value a single binary with proxy + schema registry built in, and can dedicate cores/hugepages.

**Avoid when:**
- You require a fully OSI-open-source broker with no source-available/enterprise split — Apache Kafka or NATS fit better.
- You run tiny, bursty, or heavily oversubscribed workloads where thread-per-core buys nothing.
- You depend on the deep, mature JVM Kafka operator tooling and Cruise Control-style ecosystem you already run.

## Alternatives

- apache/kafka — the reference implementation; use it when you want the canonical broker, the full JVM ecosystem, and true Apache licensing (KRaft already removes ZooKeeper).
- apache/pulsar — use it when you need tiered storage, geo-replication, and multi-tenancy as first-class, and can accept a broker/BookKeeper split.
- nats-io/nats-server — use it when you want a lightweight, genuinely open-source messaging + JetStream log without the Kafka protocol surface.
- automq/automq — use it when you want a Kafka-compatible broker built S3-first/diskless for cloud elasticity.
- confluentinc/confluent-kafka-go — not a broker, but the client path if you standardize on Confluent's Kafka distribution instead.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-11 | Repository opened publicly by Vectorized; Kafka-compatible C++ broker on Seastar[^2]. |
| — | 2022 | Company renamed Vectorized → Redpanda Data; calendar versioning adopted. |
| 23.x | 2023 | WASM Data Transforms and tiered-storage maturation. |
| 24.x | 2024 | Iceberg/analytics and enterprise feature expansion. |
| 25.2.x | 2025 | Current stable line at time of writing[^3]. |

## References

[^1]: Redpanda licensing — Business Source License 1.1 and Redpanda Community License. https://github.com/redpanda-data/redpanda/blob/dev/licenses/bsl.md and https://github.com/redpanda-data/redpanda/blob/dev/licenses/rcl.md
[^2]: Redpanda README and project overview. https://github.com/redpanda-data/redpanda
[^3]: Redpanda releases (calendar versioning, `YY.MAJOR.PATCH`). https://github.com/redpanda-data/redpanda/releases

## Tags

streaming, kafka-compatible, message-broker, cpp, seastar, raft, event-streaming, distributed-systems, real-time, source-available, tiered-storage
