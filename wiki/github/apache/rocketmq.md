# apache/rocketmq

> A distributed messaging and streaming platform, originally built at Alibaba, tuned for financial-grade transactional messaging and very deep queue accumulation.

[GitHub repo](https://github.com/apache/rocketmq) ·
[Official website](https://rocketmq.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/rocketmq/blob/develop/LICENSE)

## Overview

RocketMQ is a JVM-based message broker that started as Alibaba's internal system (the lineage runs through an earlier product often referred to as MetaQ) and was donated to the Apache Software Foundation in late 2016, graduating to a top-level project in 2017[^1]. It was designed under e-commerce load — trillions of messages during peak sale events — so its priorities are ordered delivery, transactional guarantees, and the ability to let consumers fall far behind without collapsing the broker.

The defining architectural choice is the **CommitLog**: every message on a broker, across every topic and queue, is appended to one shared log file, and per-queue *ConsumeQueue* index files point back into it[^2]. This gives near-sequential disk writes regardless of topic count, which is why RocketMQ tolerates thousands of topics far better than a partition-per-topic design. The cost is that read performance and tail latency are bound to the OS page cache and to how the single log flushes.

The other defining tension is the **4.x vs 5.x split**. The 4.x line is the mature, remoting-protocol, client-does-the-work design that most production deployments still run. The 5.x line (GA 2022) adds a stateless Proxy tier, gRPC-based clients, server-side "Pop" consumption, a Raft-style Controller for automatic failover, and tiered storage[^3]. The two are interoperable but conceptually different systems, and choosing which to standardize on is the first real decision an operator makes.

## Getting Started

RocketMQ needs a JDK (8 or newer) and runs as separate NameServer and Broker processes[^4].

```shell
# Download and unpack a binary release (check rocketmq.apache.org/download for current version)
wget https://dist.apache.org/repos/dist/release/rocketmq/5.5.0/rocketmq-all-5.5.0-bin-release.zip
unzip rocketmq-all-5.5.0-bin-release.zip
cd rocketmq-all-5.5.0-bin-release/bin

# 1) Start the NameServer (routing registry, listens on :9876)
nohup sh mqnamesrv &

# 2) Start a Broker and register it with the NameServer
nohup sh mqbroker -n localhost:9876 &
```

A minimal producer with the classic Java client:

```java
DefaultMQProducer producer = new DefaultMQProducer("example_group");
producer.setNamesrvAddr("localhost:9876");
producer.start();

Message msg = new Message("TopicTest", "TagA",
        "hello".getBytes(StandardCharsets.UTF_8));
SendResult result = producer.send(msg);   // synchronous, returns broker + queue offset
System.out.println(result.getSendStatus());

producer.shutdown();
```

Docker (`apache/rocketmq`) and the Kubernetes operator (`apache/rocketmq-operator`) are the other supported deployment paths.

## Architecture / How It Works

Four roles, no ZooKeeper:

- **NameServer** — a stateless routing registry. Brokers push heartbeats with their topic/queue metadata; producers and consumers pull routing tables from it. NameServers do not talk to each other — you run several and clients query any of them. This is an AP, eventually-consistent design, not a consensus store.
- **Broker** — stores messages and serves reads. Brokers are grouped master/slave; a group shares a `brokerName` and is distinguished by `brokerId` (0 = master).
- **Producer** / **Consumer** — clients grouped by name. Consumers coordinate partition assignment among themselves via rebalancing, and (in the classic model) track their own progress.

**Storage.** Writes go to the append-only CommitLog via memory-mapped files. Two derived structures are built asynchronously: **ConsumeQueue** (a compact per-queue index of offset/size/tag-hash used for consumption) and **IndexFile** (a hash index enabling lookup by message key). Because consumption reads ConsumeQueue then dereferences into CommitLog, cold reads that miss the page cache turn into random disk I/O.

**Replication and failover.** 4.x offers async or sync master/slave replication, but a slave could not be auto-promoted without operator action. **DLedger** added a Raft-based CommitLog so a group could elect a new master automatically, at the cost of a majority-write quorum. 5.x's **Controller** mode decouples election from the log, allowing automatic failover while keeping the plain (non-DLedger) storage format[^3].

**Message features that are first-class, not add-ons:** two-phase **transactional messages** (send a half-message, run a local transaction, then commit/rollback via a broker callback); **ordered** messages by pinning a key to one queue (FIFO holds only within a queue); **scheduled/delayed** messages (4.x exposes 18 fixed delay levels; 5.x supports arbitrary-time delivery); **Tag** and **SQL92** server-side filtering; and a dead-letter queue after retry exhaustion.

**5.x Proxy.** The Proxy is a stateless translation and routing tier between gRPC clients and brokers. It enables the newer `rocketmq-clients` (gRPC/protobuf) and the **Pop** consumption model, where the broker owns queue assignment and offset bookkeeping, so consumers can be stateless and lightweight — a better fit for serverless and multi-language clients than the classic self-rebalancing model.

## Production Notes

**NameServer is not a source of truth.** Routing propagation is on a timer (heartbeats and client polls default to the tens-of-seconds range), so a broker that just died can still appear in routing tables briefly. Clients handle this by failing over sends to other queues, but expect a window of send failures / retries on broker loss. Never treat NameServer as a strongly-consistent config store.

**Page cache is the performance model.** Throughput and, more importantly, tail latency depend on the OS page cache staying warm and on flush behavior. Synchronous flush (`FLUSH_DISK_TYPE=SYNC_FLUSH`) trades throughput for durability; async flush risks losing the last unflushed writes on a host crash. Large `pagecache` eviction or dirty-page writeback storms show up as periodic latency spikes ("burr"). RocketMQ pre-warms and locks mapped memory to mitigate this, but it is sensitive to disk choice — SSD/NVMe strongly preferred.

**Retention deletes regardless of consumption.** CommitLog files are removed by age (default retention around 72 hours) or when disk crosses a high-water mark, *whether or not consumers have read them*. A slow or stuck consumer group can silently lose un-consumed data when the log rolls off. Monitor consumer lag and disk usage together.

**Replication choice is a real decision.** Async master/slave is fast but can lose in-flight messages on master failure. Sync (`SYNC_MASTER`) blocks the producer until the slave acks. DLedger/Controller give auto-failover but need an odd-sized quorum (typically 3 nodes) and pay Raft's write-amplification. Mixing these up is the most common durability surprise.

**4.x → 5.x is a migration, not an upgrade.** Brokers can run in a compatible mode and old remoting clients keep working, but adopting the Proxy, gRPC clients, Pop, and Controller means new components to operate and new failure modes. Many shops stay on the 4.9.x line precisely because it is a smaller, well-understood system.

**Operational surface is large.** NameServers, master/slave broker groups, optionally a Controller/DLedger quorum, optionally Proxies, plus the Dashboard for visibility. It is more moving parts than a single-binary broker, and the docs — improved but historically thin and partly Chinese-first — are a real friction point for teams outside the original community.

## When to Use / When Not

**Use when:**
- You need transactional (two-phase) messaging or strict per-key ordering as a built-in, not a bolt-on.
- You have very large numbers of topics and cannot afford Kafka's partition-count overhead.
- You expect consumers to accumulate deep backlogs and replay by time or offset.
- You are already in a JVM/Alibaba-cloud-adjacent ecosystem or want a broker with financial-grade messaging semantics.

**Avoid when:**
- You want a single-binary, low-ops broker — RabbitMQ, NATS, or Redis Streams are far simpler to run.
- Your workload is log/stream analytics with a mature connector and processing ecosystem — Kafka's ecosystem (Connect, Streams, ksqlDB, the whole vendor landscape) is deeper.
- You need a broad, polyglot client story with first-class non-JVM SDKs; the classic clients are JVM-centric and the gRPC clients are newer.
- You cannot invest in operating multiple coordinated components and tuning page-cache/flush behavior.

## Alternatives

- apache/kafka — use instead when you want the dominant streaming ecosystem (Connect, Streams, tiered storage, huge vendor/tooling support) and partition-based log semantics.
- apache/pulsar — use instead when you want compute/storage separation (BookKeeper), native multi-tenancy, and geo-replication out of the box.
- rabbitmq/rabbitmq-server — use instead when you want flexible routing (exchanges, AMQP) and simple ops over deep accumulation and streaming.
- nats-io/nats-server — use instead when you want a tiny, fast, cloud-native messaging core (with JetStream for persistence) and minimal operational weight.
- redis/redis (Streams) — use instead when you already run Redis and need lightweight streams without a dedicated broker tier.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-11 | Donated to the Apache Incubator by Alibaba[^1]. |
| — | 2017-09 | Graduated to an Apache Top-Level Project[^1]. |
| 4.0 | 2017 | First Apache 4.x line; remoting protocol, classic clients. |
| 4.5 | 2019 | DLedger (Raft-based CommitLog) for automatic master failover[^2]. |
| 4.9.x | 2021+ | Long-lived, widely deployed 4.x maintenance line. |
| 5.0.0 | 2022-09 | New architecture: Proxy tier, gRPC clients, Pop, Controller, tiered storage[^3]. |
| 5.5.0 | 2024 | Release referenced by the project's current quick-start[^4]. |

## References

[^1]: Apache RocketMQ project history / ASF incubation and graduation. https://rocketmq.apache.org/about/
[^2]: RocketMQ design docs — storage model (CommitLog, ConsumeQueue, IndexFile) and DLedger. https://rocketmq.apache.org/docs/
[^3]: Apache RocketMQ 5.0 architecture (Proxy, Pop, Controller, tiered storage). https://rocketmq.apache.org/docs/
[^4]: Apache RocketMQ README and Quick Start. https://github.com/apache/rocketmq/blob/develop/README.md
[^5]: Apache RocketMQ download page (current releases). https://rocketmq.apache.org/download/

## Tags

java, jvm, message-queue, messaging, streaming, event-driven, distributed-systems, transactional-messaging, apache, broker, cloud-native, pub-sub
