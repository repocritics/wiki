# rabbitmq/rabbitmq-server

> Erlang-based multi-protocol message broker — the reference AMQP 0-9-1 implementation, now a general-purpose messaging and streaming server.

[GitHub repo](https://github.com/rabbitmq/rabbitmq-server) ·
[Official website](https://www.rabbitmq.com/) ·
[License: MPL-2.0](https://github.com/rabbitmq/rabbitmq-server/blob/main/LICENSE-MPL-RabbitMQ)

## Overview

RabbitMQ is a message broker written in Erlang/OTP, first released in 2007 by Rabbit Technologies as the reference implementation of the AMQP 0-9-1 protocol[^1]. It has since absorbed several other wire protocols — AMQP 1.0, MQTT (3.1 through 5.0), STOMP, and its own binary Stream protocol — behind a single clustered server. The project passed through SpringSource, VMware, and Pivotal, and is now maintained by Broadcom following its 2023 acquisition of VMware; the copyright header reads Broadcom, and commercial builds ship as Tanzu RabbitMQ. (GitHub's language detector reports JavaScript because of the management-UI plugin; the server itself is Erlang.)

The defining tension in RabbitMQ is **flexible routing versus operational simplicity**. Its exchange/binding/queue model (direct, topic, fanout, headers exchanges) makes it the most expressive routing broker in common use — far richer than a Kafka topic or an SQS queue. That same flexibility, combined with Erlang's clustering model and a decade of accumulated queue types, means the operational surface is large: the wrong queue type, mirroring mode, or partition-handling setting can silently lose messages under failure. Much of RabbitMQ's version history since 3.8 has been about narrowing that surface — replacing unsafe defaults with Raft-based replication.

RabbitMQ is a good fit when you need per-message routing, low-to-moderate throughput with strong delivery semantics, and protocol flexibility. It is a poor fit as a high-throughput event log — that is Kafka's territory, and RabbitMQ's own Streams feature is a partial, deliberately narrower answer to it.

## Getting Started

```bash
# Docker — includes the management UI plugin
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:4-management
# management UI at http://localhost:15672  (guest/guest, localhost only)
```

Minimal publish/consume with the Python `pika` client (AMQP 0-9-1):

```python
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
ch = conn.channel()
ch.queue_declare(queue="tasks", durable=True)          # survive broker restart

ch.basic_publish(
    exchange="",                                        # default direct exchange
    routing_key="tasks",                                # routes to same-named queue
    body="hello",
    properties=pika.BasicProperties(delivery_mode=2),   # persistent message
)

def handle(ch, method, props, body):
    print(body)
    ch.basic_ack(method.delivery_tag)                   # manual ack — no auto-loss

ch.basic_qos(prefetch_count=10)                         # bound unacked in flight
ch.basic_consume(queue="tasks", on_message_callback=handle)
ch.start_consuming()
```

## Architecture / How It Works

**Routing model.** Publishers send to an *exchange*, never directly to a queue. Bindings connect exchanges to queues by routing key or pattern. Exchange types — direct, topic, fanout, headers — determine matching. This indirection is RabbitMQ's core value: routing topology is broker-side configuration, not producer code.

**Queue types** are the axis that matters most operationally:

- **Classic queues** — the original single-node queue. Fast, but not replicated on their own.
- **Quorum queues** — Raft-replicated, disk-based, consistency-oriented. Introduced in 3.8 (2019) and now the recommended replicated queue[^2]. They tolerate node loss with a majority quorum and support poison-message handling via `delivery-limit`.
- **Streams** — an append-only, replicated log with non-destructive (offset-based) consumers, added in 3.9 (2021)[^3]. This is RabbitMQ's Kafka-shaped answer: consumers read by offset and re-read history, unlike the destructive semantics of a normal queue.

**Clustering** uses Erlang's built-in distribution: all nodes share definitions (users, vhosts, exchanges, bindings) and each queue/stream has a home node plus replicas. The metadata store was historically **Mnesia**; RabbitMQ 4.0 (2024) makes **Khepri** — a Raft-backed store — the default, specifically to fix Mnesia's poor behavior under network partitions[^4].

**Plugins.** Much of what people call "RabbitMQ" is tier-1 plugins shipped in this repo: `rabbitmq_management` (the HTTP API and web UI), `rabbitmq_mqtt`, `rabbitmq_stomp`, `rabbitmq_federation`, `rabbitmq_shovel`, and the Prometheus exporter. The core is a protocol-agnostic queue/exchange engine; the protocol adapters are plugins on top.

## Production Notes

**Classic mirrored queues are gone.** For years, high availability meant classic *mirrored* queues (`ha-mode` policies). These were unsafe under network partitions — Jepsen analysis demonstrated acknowledged-message loss on partition recovery[^5]. They were deprecated and **removed in RabbitMQ 4.0**. If you are on an older cluster relying on `ha-mode`, migration to quorum queues is a prerequisite for the 4.x upgrade, and it is not automatic — queue type is fixed at declaration.

**Network partitions.** Because clustering rides Erlang distribution, a partition can split the cluster. The `cluster_partition_handling` setting (`pause_minority`, `autoheal`, `ignore`) governs recovery; `ignore` plus mirrored queues was the classic footgun. Quorum queues and Khepri move consistency decisions to Raft, which is the main reason to prefer them.

**Flow control and memory alarms.** RabbitMQ applies back-pressure when memory or disk cross a watermark (`vm_memory_high_watermark`, default 0.4 of RAM; disk free limit). When triggered, publishers are blocked (`connection.blocked`) — a healthy mechanism that surprises teams who read it as an outage. Unbounded queues from slow/absent consumers are the usual root cause; set `prefetch_count`, TTLs, or max-length policies.

**Erlang coupling.** The server is tied to specific Erlang/OTP version ranges per release series[^6]. Upgrading RabbitMQ often means upgrading Erlang; check the supported-Erlang matrix before any bump.

**Throughput ceiling.** RabbitMQ is not a log. Single-queue throughput is bounded by that queue's home node, and quorum queues trade throughput for replication safety. For millions of messages per second across partitions, Kafka or Pulsar are the right tools; use RabbitMQ Streams only when you want log semantics inside an existing RabbitMQ deployment.

**Upgrades.** Feature flags (since 3.8) gate schema changes so mixed-version clusters can upgrade node-by-node — but only within supported version steps. Skipping minor series or running long-lived mixed-version clusters is unsupported and a common cause of failed rolling upgrades.

## When to Use / When Not

**Use when:**
- You need rich per-message routing (topic/headers exchanges, RPC, work queues).
- You want multiple protocols (AMQP, MQTT, STOMP) against one broker.
- You need strong per-message delivery guarantees at moderate throughput.
- You want replicated queues with a majority-quorum consistency story (quorum queues).

**Avoid when:**
- You need a high-throughput, replayable event log — that is Kafka/Pulsar, or at most RabbitMQ Streams.
- You want a fully managed, zero-ops queue and are on one cloud — SQS/Pub/Sub remove the broker entirely.
- Your team cannot own Erlang-cluster operations (partitions, alarms, upgrades).
- You only need simple background jobs backed by a database you already run.

## Alternatives

- apache/kafka — use instead when you need a replayable, partitioned event log at high throughput rather than per-message routing.
- apache/pulsar — use instead when you want log semantics plus multi-tenancy and tiered storage in one system.
- nats-io/nats-server — use instead when you want a lightweight, operationally simple broker (JetStream for persistence) without AMQP's routing model.
- redis/redis — use instead when Streams/Pub-Sub in a store you already run is enough and you don't need broker-grade durability.
- Amazon SQS/SNS — use instead when you're on AWS and want a managed queue with no broker to operate.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2007-02 | First release; reference AMQP 0-9-1 broker[^1]. |
| 3.0 | 2012-11 | Topic-exchange overhaul, plugin system maturity. |
| 3.6 | 2015-12 | Lazy queues (disk-first) for large backlogs. |
| 3.7 | 2017-11 | New config format, `rabbitmqctl` overhaul. |
| 3.8 | 2019-10 | Quorum queues, feature flags, single active consumer[^2]. |
| 3.9 | 2021-07 | Streams and the Stream protocol[^3]. |
| 3.12 | 2023-06 | Classic mirrored queues deprecated; classic queue v2 store. |
| 3.13 | 2024-03 | Khepri available (experimental); MQTT 5.0. |
| 4.0 | 2024-09 | Khepri default, mirrored queues removed, AMQP 1.0 as core protocol[^4]. |

## References

[^1]: RabbitMQ, "A history of RabbitMQ." https://www.rabbitmq.com/docs/#history — first released 2007 as an AMQP 0-9-1 broker.
[^2]: RabbitMQ docs, "Quorum Queues." https://www.rabbitmq.com/docs/quorum-queues
[^3]: RabbitMQ docs, "Streams." https://www.rabbitmq.com/docs/streams
[^4]: RabbitMQ blog, "RabbitMQ 4.0 release." https://www.rabbitmq.com/blog/2024/08/23/rabbitmq-4.0-release — Khepri default, mirrored queues removed.
[^5]: Kyle Kingsbury, "Jepsen: RabbitMQ" — acknowledged-message loss with mirrored queues under partition. https://aphyr.com/posts/315-jepsen-rabbitmq
[^6]: RabbitMQ docs, "Supported Erlang versions." https://www.rabbitmq.com/docs/which-erlang

## Tags

erlang, message-broker, amqp, mqtt, messaging, queue, streaming, distributed-systems, pub-sub, infrastructure
