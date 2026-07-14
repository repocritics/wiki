# apache/kafka

> A distributed, partitioned, replicated commit log presented as a publish-subscribe messaging system — the default backbone for event streaming.

[GitHub repo](https://github.com/apache/kafka) ·
[Official website](https://kafka.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/kafka/blob/trunk/LICENSE)

## Overview

Kafka is a distributed event streaming platform: producers append records to
partitioned, append-only logs (topics), and consumers read from them at their own
pace by tracking an offset. It was built at LinkedIn to move high-volume activity
and operational data, open-sourced in 2011, and became a top-level Apache project
in 2012[^1]. The codebase is Java and Scala; the broker and most tooling run on the
JVM, with clients available in most languages.

The defining design choice is that Kafka is a **log, not a queue**. Messages are
not deleted when consumed — they persist for a retention window (time- or
size-bounded), and multiple independent consumer groups can replay the same
partition. Ordering is guaranteed only *within* a partition, and a partition is the
unit of parallelism, replication, and ordering all at once. This single decision is
the source of most of Kafka's strengths (throughput, replayability, multiple
readers) and most of its operational sharp edges (partition count is hard to change,
keys must be chosen carefully, cross-partition ordering does not exist).

Kafka ships three layers that are often conflated: the **broker/core** (the log and
replication protocol), **Kafka Connect** (a framework for source/sink connectors to
external systems), and **Kafka Streams** (a JVM library for stateful stream
processing on top of the consumer API). Connect and Streams are optional; a large
share of deployments use only the core broker as a durable buffer between systems.

## Getting Started

Kafka 4.x requires no external ZooKeeper — the broker runs in KRaft mode. Quickest
path is the official Docker image[^2]:

```bash
docker run -p 9092:9092 apache/kafka:latest
```

Produce and consume with the bundled CLI tools:

```bash
# create a topic with 3 partitions
bin/kafka-topics.sh --create --topic orders --partitions 3 \
  --bootstrap-server localhost:9092

# produce (type lines, Ctrl-D to end)
bin/kafka-console-producer.sh --topic orders --bootstrap-server localhost:9092

# consume from the beginning
bin/kafka-console-consumer.sh --topic orders --from-beginning \
  --bootstrap-server localhost:9092
```

A minimal Java producer using the official client:

```java
var props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all");           // wait for all in-sync replicas
props.put("enable.idempotence", "true");

try (var producer = new KafkaProducer<String, String>(props)) {
    // key decides the partition -> ordering is per-key
    producer.send(new ProducerRecord<>("orders", "user-42", "order placed"));
}
```

## Architecture / How It Works

A Kafka cluster is a set of **brokers**. Each topic is split into **partitions**;
each partition is an ordered, immutable log stored as segment files on disk. Every
partition has one **leader** broker and zero or more **follower** replicas. Producers
and consumers talk only to the leader; followers pull to stay in sync.

- **Replication and ISR.** The set of replicas caught up to the leader is the
  in-sync replica set (ISR). `acks=all` combined with `min.insync.replicas` defines
  the durability contract: a write is acknowledged only once enough replicas have it.
  Set `min.insync.replicas=2` with replication factor 3 to survive one broker loss
  without data loss.
- **Consumer groups.** Consumers in a group divide partitions among themselves; each
  partition is read by exactly one member. Adding consumers scales reads up to the
  partition count and no further. Group membership changes trigger a **rebalance**;
  modern clients use incremental cooperative rebalancing to avoid stop-the-world
  pauses.
- **Offsets.** Consumer progress is stored in the internal `__consumer_offsets` topic,
  not on the broker per-consumer. This is why replay is just "seek to an earlier
  offset."
- **KRaft.** Cluster metadata (topics, partitions, ISR, configs) historically lived
  in ZooKeeper. KIP-500 replaced it with **KRaft**, a self-managed Raft quorum of
  controller nodes storing metadata in an internal Kafka log. KRaft reached
  production readiness in 3.3, and ZooKeeper was removed entirely in 4.0[^3].
- **Performance.** Throughput comes from sequential disk writes, the OS page cache
  (Kafka deliberately does not maintain its own message cache), and zero-copy
  transfer (`sendfile`) from page cache to socket. This is why Kafka is memory- and
  I/O-bound far more than CPU-bound.
- **Exactly-once.** The idempotent producer plus the transactions API (since 0.11)
  provide exactly-once semantics across a read-process-write cycle within Kafka.
  End-to-end EOS to external systems still depends on the sink being idempotent or
  transactional.

## Production Notes

**Partition count is a semi-permanent decision.** You can increase partitions but
never decrease them, and increasing them breaks key-to-partition mapping (a given
key may land on a different partition afterward), silently violating per-key
ordering for keyed topics. Size partitions up front for peak parallelism, not
current load.

**Rebalances are the classic outage.** A slow consumer that misses
`max.poll.interval.ms` gets evicted, triggering a rebalance that stalls the whole
group. Long processing per record, large `max.poll.records`, and GC pauses all feed
this. Tune poll intervals and prefer cooperative rebalancing.

**Watch under-replicated and offline partitions.** `UnderReplicatedPartitions > 0`
means a replica is lagging or a broker is down; `OfflinePartitionsCount > 0` means a
partition has no leader and is unavailable. Consumer lag (per-group, per-partition)
is the other essential signal — it is the real "are we falling behind" metric.

**Page cache is load-bearing.** Give the OS plenty of free RAM; do not oversize the
JVM heap at the page cache's expense. Co-locating other memory-hungry processes on
brokers degrades throughput indirectly by evicting Kafka's cache.

**Hot partitions.** Poor key distribution (e.g., a null or low-cardinality key)
funnels traffic to one partition and one broker, so cluster-wide capacity does not
help. Rebalancing partition placement across brokers is a manual/tooling exercise
(`kafka-reassign-partitions.sh`, Cruise Control).

**Upgrades are rolling but protocol-gated.** Historically the
`inter.broker.protocol.version` had to be bumped only after all brokers were on the
new binary; getting the order wrong stalls upgrades. The ZooKeeper-to-KRaft
migration path (for clusters predating 4.0) is a multi-step operational project, not
a version bump.

**Tiered storage** (KIP-405) offloads older log segments to object storage, letting
brokers keep long retention without local disk growth — useful, but changes the
latency profile for historical reads and adds an external dependency.

## When to Use / When Not

**Use when:**
- You need a durable, replayable buffer decoupling producers from many independent
  consumers.
- High sustained throughput (hundreds of MB/s per cluster) matters and per-partition
  ordering is sufficient.
- You want event sourcing, CDC pipelines, or a log that multiple teams read
  independently.
- You need stream processing co-located with the log (Kafka Streams / ksqlDB).

**Avoid when:**
- You need per-message routing, priorities, or per-message acknowledgement/redelivery
  — a traditional broker (RabbitMQ) fits better.
- You have low volume and want minimal ops — a single Kafka cluster is a lot of
  machinery for a small queue.
- You need strict global ordering across all messages — Kafka only orders within a
  partition.
- Request/reply RPC semantics — Kafka is fire-and-forget log append, not RPC.

## Alternatives

- apache/pulsar — use when you want segment-based storage decoupled from brokers, native multi-tenancy, and built-in tiered storage.
- redpanda-data/redpanda — use when you want the Kafka protocol without the JVM or ZooKeeper; a C++ single-binary broker tuned for tail latency.
- nats-io/nats-server — use when you want lightweight messaging with optional JetStream persistence and far simpler operations.
- rabbitmq/rabbitmq-server — use when you need per-message routing, acknowledgements, priorities, and classic queue semantics rather than a replayable log.
- apache/rocketmq — use when you want a broker with first-class ordered/transactional/delayed messaging, popular in the Alibaba ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.7 | 2012 | First Apache release after LinkedIn open-sourced it[^1]. |
| 0.8 | 2013-12 | Intra-cluster replication (the durability model). |
| 0.9 | 2015-11 | New consumer, security (SASL/TLS), Kafka Connect. |
| 0.10 | 2016-05 | Kafka Streams, record timestamps. |
| 0.11 | 2017-06 | Idempotent producer + transactions (exactly-once). |
| 1.0 | 2017-11 | API maturity milestone. |
| 2.0 | 2018-07 | Incremental cooperative rebalancing groundwork, security hardening. |
| 3.0 | 2021-09 | KRaft preview; ZooKeeper deprecation path begins[^3]. |
| 3.3 | 2022-10 | KRaft marked production-ready. |
| 3.6 | 2023-10 | Tiered storage (early access, KIP-405). |
| 4.0 | 2025 | ZooKeeper removed; KRaft-only broker[^3]. |

## References

[^1]: Apache Kafka — project history and origins at LinkedIn. https://kafka.apache.org/intro
[^2]: Apache Kafka Docker image and quickstart. https://kafka.apache.org/quickstart
[^3]: KIP-500: Replace ZooKeeper with a self-managed metadata quorum (KRaft). https://cwiki.apache.org/confluence/display/KAFKA/KIP-500

## Tags

java, scala, event-streaming, message-broker, distributed-systems, commit-log, pub-sub, kafka, streaming, data-pipeline, kraft
