# apache/pulsar

> Distributed pub-sub and queuing messaging platform with a broker/storage split — brokers are stateless, durability lives in Apache BookKeeper.

[GitHub repo](https://github.com/apache/pulsar) ·
[Official website](https://pulsar.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/pulsar/blob/master/LICENSE)

## Overview

Apache Pulsar is a distributed messaging system that combines streaming (Kafka-style, replayable logs) and queuing (RabbitMQ-style, per-message acknowledgement) in one broker. It originated at Yahoo, was open-sourced in 2016, entered the Apache Incubator in 2017, and graduated to a top-level project in September 2018[^1]. It is written in Java and, as of 2026, is one of the two mainstream open-source event-streaming platforms alongside Kafka.

The defining architectural choice is the **separation of serving from storage**. Brokers hold no persistent data; they are a stateless serving tier. Durable message storage is delegated to **Apache BookKeeper**, a separate distributed write-ahead-log system, and cluster metadata lives in ZooKeeper (or, in newer versions, Oxia). This decoupling is Pulsar's central selling point and its central complexity: you can scale serving and storage independently and rebalance topics between brokers in milliseconds (no data moves), but you operate at least three distributed systems instead of one.

Pulsar was built for multi-tenancy from the start. Its topic namespace is hierarchical — `persistent://tenant/namespace/topic` — and authentication, authorization, quotas, and isolation policies attach at the tenant and namespace level. This makes it a natural fit for a shared internal messaging service across many teams, which is close to the problem Yahoo built it to solve.

## Getting Started

Run a single-node standalone instance (bundles broker, BookKeeper, and ZooKeeper in one process — for local development only):

```bash
docker run -it -p 6650:6650 -p 8080:8080 \
  apachepulsar/pulsar:latest \
  bin/pulsar standalone
```

A minimal Java producer/consumer:

```java
PulsarClient client = PulsarClient.builder()
    .serviceUrl("pulsar://localhost:6650")
    .build();

Producer<byte[]> producer = client.newProducer()
    .topic("persistent://public/default/my-topic")
    .create();
producer.send("hello".getBytes());

Consumer<byte[]> consumer = client.newConsumer()
    .topic("persistent://public/default/my-topic")
    .subscriptionName("my-sub")
    .subscriptionType(SubscriptionType.Shared)   // queue semantics
    .subscribe();

Message<byte[]> msg = consumer.receive();
consumer.acknowledge(msg);
```

The subscription type — `Exclusive`, `Failover`, `Shared`, or `Key_Shared` — is where Pulsar's dual nature lives: `Shared` gives you a work queue with round-robin dispatch and per-message acks; `Failover`/`Exclusive` give you ordered log-style consumption.

## Architecture / How It Works

Three layers, deployed as three separate clusters:

1. **Brokers** — stateless serving tier. A broker owns a topic (holds its dispatch state and cursor positions) but persists nothing locally. Topic ownership is assigned by a load manager and can move between brokers instantly because there is no data to migrate.
2. **BookKeeper (bookies)** — the durable storage tier. Messages are written to append-only **ledgers**; a topic is a sequence of ledgers (a "managed ledger"). Writes are replicated across `N` bookies with configurable write/ack quorum, so durability is a BookKeeper property, not a broker one.
3. **Metadata store** — ZooKeeper historically, holding cluster config, ownership, and BookKeeper metadata. Newer versions add **Oxia** as a scalable ZooKeeper alternative, and a longer-term goal is to reduce the hard ZooKeeper dependency[^2].

Because storage is **segment-oriented** rather than partition-oriented, a single topic's backlog can exceed any one node's disk — it spreads across bookies as new ledgers roll. This is the structural difference from Kafka, where a partition is pinned to a broker's local disk and rebalancing means copying data.

Other subsystems:

- **Cursors** — subscription progress (the ack position) is stored durably in BookKeeper, not inferred from an offset the client tracks. Individual (out-of-order) acks are supported, which is what enables true queue semantics.
- **Tiered storage** — older ledgers can be offloaded to S3/GCS/HDFS and read back transparently, so a topic can retain effectively unbounded history at object-storage cost.
- **Pulsar Functions** — a lightweight serverless compute framework that runs consume-process-produce functions inside or alongside brokers. **Pulsar IO** connectors are Functions under the hood.
- **Geo-replication** — configured per namespace; brokers asynchronously (or synchronously) replicate across clusters.
- **Protocol handlers** — KoP (Kafka-on-Pulsar), MoP (MQTT), and AoP (AMQP) let Pulsar speak other wire protocols, with varying maturity.

## Production Notes

**You are operating three distributed systems.** The broker/BookKeeper/ZooKeeper split is real operational surface. BookKeeper has its own tuning (journal vs. ledger device separation, `ensemble`/`write`/`ack` quorum sizing), ZooKeeper has its own failure modes, and diagnosing a latency spike means knowing which tier it came from. Teams routinely underestimate this; a "just messaging" mental model does not survive first contact.

**Backlog and storage quotas are load-bearing.** Because retention is cheap and durable, an unacknowledged subscription silently accumulates backlog that pins ledgers from being deleted — one abandoned consumer can grow storage without bound. Namespace-level backlog quotas and retention/TTL policies are not optional in production; they are the primary guardrail.

**BookKeeper disk layout matters.** For write-heavy workloads, the journal (sync write path) and the ledger storage device should be physically separate disks. Co-locating them is a common cause of latency that gets misattributed to the broker.

**ZooKeeper is a scaling ceiling.** At very high topic counts (hundreds of thousands to millions), ZooKeeper metadata operations and broker load-balancing become the bottleneck before raw throughput does. The Oxia work exists precisely to raise this ceiling; check whether your version and your metadata backend match your topic scale.

**Client/broker version skew.** The wire protocol is generally backward compatible, but Java runtime requirements are strict per version: Pulsar 4.0+ brokers require **JDK 21**, and older lines required 17 or 11[^3]. The build system migrated from Maven to **Gradle** in recent releases, which matters if you build from source or maintain forks.

**Functions are convenient, not free.** Running Pulsar Functions in the broker process couples compute failures to your serving tier; the process/Kubernetes runtimes isolate them but add deployment complexity. For anything beyond trivial transforms, an external stream processor is often the better boundary.

## When to Use / When Not

**Use when:**
- You genuinely need both streaming and queuing semantics from one system (replayable logs *and* per-message-ack work queues).
- You are building a multi-tenant messaging service shared across many teams — tenancy and isolation are first-class, not bolted on.
- You want to retain long or unbounded history cheaply via tiered object storage.
- You need very high topic counts and fast, data-free topic rebalancing.
- Geo-replication across regions is a core requirement rather than an afterthought.

**Avoid when:**
- You want the smallest possible operational footprint — a single Kafka-compatible service (or a managed queue like SQS) is far less to run.
- Your team lacks the appetite to operate BookKeeper and ZooKeeper as first-class systems.
- The surrounding ecosystem you depend on (connectors, stream processors, tooling, hiring pool) is Kafka-shaped — Pulsar's ecosystem is real but smaller.
- Your workload is a simple task queue — Pulsar is a lot of machinery for that.

## Alternatives

- apache/kafka — the dominant log-based streaming platform; larger ecosystem and hiring pool, but partitions are pinned to broker disks so rebalancing copies data. Use it when ecosystem gravity and operational familiarity outweigh Pulsar's storage flexibility.
- redpanda-data/redpanda — Kafka-API-compatible broker in C++ with no JVM/ZooKeeper/BookKeeper; use it when you want Kafka semantics with a single-binary, lower-latency operational profile.
- nats-io/nats-server — much lighter, Go, with JetStream persistence; use it when you want simple, low-latency messaging without the three-tier operational cost.
- rabbitmq/rabbitmq-server — mature broker for classic queuing/routing; use it when you need rich routing topologies and per-message queue semantics but not log replay at scale.
- apache/rocketmq — comparable unified streaming/queuing system with strong adoption in China; use it when its ecosystem or ordering model fits better.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-sourced | 2016-09 | Released by Yahoo[^1]. |
| incubating | 2017-06 | Entered the Apache Incubator. |
| TLP | 2018-09 | Graduated to Apache top-level project[^1]. |
| 2.0 | 2018-06 | Pulsar Functions, schema registry, topic compaction. |
| 2.x line | 2019–2023 | Tiered storage, transactions, protocol handlers (KoP/MoP), broad ecosystem growth. |
| 3.0 | 2023-05 | First designated Long-Term-Support release; JDK 17 baseline. |
| 4.0 | 2024-10 | LTS; JDK 21 requirement, Oxia metadata option, continued ZooKeeper-reduction work[^2][^3]. |

## References

[^1]: Apache Software Foundation, "The Apache Software Foundation Announces Apache Pulsar as a Top-Level Project" — 2018-09-25. https://blogs.apache.org/foundation/entry/the-apache-software-foundation-announces58
[^2]: Apache Pulsar / Oxia — scalable metadata store intended as a ZooKeeper alternative. https://github.com/apache/oxia
[^3]: Apache Pulsar README, "Pulsar Runtime Java Version Recommendation" and build requirements (JDK 21 for 4.0+). https://github.com/apache/pulsar/blob/master/README.md

## Tags

java, messaging, pub-sub, event-streaming, distributed-systems, message-queue, apache, bookkeeper, geo-replication, multi-tenant
