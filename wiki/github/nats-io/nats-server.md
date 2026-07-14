# nats-io/nats-server

> Subject-based messaging system with fire-and-forget pub/sub at its core and an opt-in persistence layer (JetStream) bolted on top.

[GitHub repo](https://github.com/nats-io/nats-server) ·
[Official website](https://nats.io) ·
[License: Apache-2.0](https://github.com/nats-io/nats-server/blob/main/LICENSE)

## Overview

NATS is a messaging server written in Go. It was created by Derek Collison — originally in Ruby as the message bus inside Cloud Foundry, then rewritten in Go — and is now maintained by Synadia and a CNCF community[^1]. The core of NATS is deliberately minimal: a text-based TCP protocol, subject-based addressing with wildcards, and **at-most-once** delivery. A published message that has no interested subscriber at that instant is simply dropped. There is no broker-side queue, no acknowledgement, and no persistence in core NATS.

That minimalism is the defining tension. Core NATS is fast and operationally simple because it does almost nothing on the server side — it is closer to a routing fabric than a message broker. Everything people associate with "real" messaging (durable queues, replay, exactly-once, dead-lettering) lives in **JetStream**, a separate persistence subsystem added in 2021 that runs on the same server and speaks the same protocol[^2]. Deciding which half of the product you are actually using — the stateless fabric or the stateful log — is the single most important architectural choice, and the source of most confusion for newcomers who expect Kafka-like semantics from the base layer.

NATS targets microservice-to-microservice communication, edge/IoT fan-in (it runs on a Raspberry Pi and supports MQTT and WebSocket clients), and multi-tenant platforms. There are 40+ client language implementations. Its differentiators versus Kafka/RabbitMQ are subject wildcards, built-in request-reply, and a topology (leaf nodes, superclusters) designed for geographically distributed and edge deployments.

## Getting Started

```bash
# Run a server (Docker)
docker run -p 4222:4222 nats:latest

# Or install the binary
go install github.com/nats-io/nats-server/v2@latest

# Enable JetStream for persistence
nats-server -js
```

Minimal Go publisher/subscriber using the `nats.go` client:

```go
package main

import (
	"fmt"
	"github.com/nats-io/nats.go"
)

func main() {
	nc, _ := nats.Connect("nats://localhost:4222")
	defer nc.Drain()

	// Subscribe to a subject with a wildcard token.
	nc.Subscribe("orders.*", func(m *nats.Msg) {
		fmt.Printf("%s: %s\n", m.Subject, string(m.Data))
	})

	// Fire-and-forget publish — dropped if nobody is listening.
	nc.Publish("orders.created", []byte("order-42"))
	nc.Flush()
}
```

Request-reply (a first-class pattern, not an add-on) is `nc.Request(subject, data, timeout)` on one side and a subscriber that calls `m.Respond(...)`.

## Architecture / How It Works

**Subjects and wildcards.** Addressing is hierarchical dot-delimited subjects (`orders.us.created`). Subscribers match with `*` (single token) and `>` (one-or-more trailing tokens). There are no pre-declared topics — subjects are ephemeral and created implicitly by publishing. **Queue groups** turn a set of subscribers on the same subject into a load-balanced pool (one member gets each message), which is how NATS does work distribution without a durable queue.

**Clustering topology.** Servers form a full-mesh **cluster** that gossips subscription interest so a message only crosses a link if a remote subscriber exists. Clusters connect to other clusters through **gateways** to form a **supercluster** (spanning regions). **Leaf nodes** are edge servers that connect outward to a hub cluster, carrying their own local subject space — this is the mechanism behind NATS's edge/IoT story.

**Security and multi-tenancy.** NATS 2.0 introduced **accounts**: isolated subject namespaces within one server, with explicit import/export of subjects between accounts. Decentralized auth uses **nkeys** (Ed25519 keypairs) and a JWT hierarchy of operator → account → user, so the server can authorize clients without a shared credential store.

**JetStream.** The persistence layer stores messages in **streams** (append-only logs bound to a set of subjects) on file or memory storage. Consumers pull or push messages with explicit acks, redelivery, and replay from a sequence/time. Replication across a cluster uses **Raft** for quorum, so replica counts are R1/R3/R5 and a cluster needs an odd number of nodes for a healthy quorum. The **KV store** and **Object store** are thin abstractions built on top of JetStream streams — a KV bucket is a stream with last-value-per-subject retention.

The whole system multiplexes over one protocol and one port range, so MQTT, WebSocket, and leaf-node traffic all terminate in the same server process. This is why NATS feels lighter to operate than a Kafka + ZooKeeper/KRaft + schema-registry stack, at the cost of fewer knobs.

## Production Notes

**Core NATS drops messages — by design.** If a subscriber is slow, the server marks it a **slow consumer** and disconnects it rather than buffering unboundedly. Teams that assume broker-side durability get silent data loss. If you need delivery guarantees, you are using JetStream, and you must design for it explicitly.

**MaxPayload is small and fixed at connect time.** Default max message size is 1 MB; it is configurable up to 64 MB but is advertised to clients at connection and cannot be exceeded. NATS is a messaging fabric, not a blob transport — large payloads belong in the Object store or external storage with a reference passed over NATS.

**JetStream sizing is the real operational work.** File storage, stream retention (limits/interest/workqueue), replica count, and per-consumer ack/redelivery all interact. High **subject cardinality** (e.g., a KV bucket with millions of distinct keys, since each key is a subject) stresses the stream index and memory. Under-provisioned disks turn JetStream into a latency source. Monitor via `nats-server` metrics and the `nats` CLI (`nats stream report`, `nats consumer report`).

**Clustering quorum.** JetStream replication is Raft-based, so an even node count or a network partition can lose quorum and stall writes for affected streams. Plan for 3 or 5 nodes and understand which streams are R3 vs R1 (R1 streams have no failover).

**Upgrade and deprecation history to know:** the older **NATS Streaming (STAN)** server was a separate product, superseded by JetStream and reaching end-of-life in 2023 — do not start new work on STAN. Within the 2.x line, JetStream defaults and clustering behavior have evolved across 2.9/2.10/2.11, so read release notes before rolling upgrades of a JetStream cluster; mixed-version clusters during upgrade are supported but version-sensitive for JetStream metadata.

**Auth complexity curve.** Simple token/user-password auth is trivial. The decentralized nkey/JWT model (operator/account/user) is powerful for multi-tenant platforms but has a steep setup cost and its own tooling (`nsc`). Choose based on tenancy needs, not defaults.

## When to Use / When Not

**Use when:**
- You need low-latency service-to-service messaging with request-reply and load-balanced queue groups.
- You have edge/IoT fan-in or geographically distributed nodes (leaf nodes, superclusters, MQTT support).
- You want one lightweight binary instead of a Kafka-class operational footprint.
- You want subject-based routing with wildcards and multi-tenant account isolation.

**Avoid when:**
- You need a durable, ordered, replayable log as the primary abstraction and don't want to operate JetStream carefully — Kafka is purpose-built for that.
- You are moving large payloads or files through the broker (payload cap; wrong tool).
- Your team expects broker-side durability by default and won't internalize core NATS's at-most-once semantics.
- You need mature exactly-once stream processing / connectors ecosystem (Kafka Connect, ksqlDB have no direct NATS equal).

## Alternatives

- apache/kafka — use instead when a durable, partitioned, replayable log with a large connector/stream-processing ecosystem is the core requirement.
- rabbitmq/rabbitmq-server — use instead when you want mature AMQP routing, per-message durability, and complex exchange topologies out of the box.
- nsqio/nsq — use instead when you want a simple distributed queue with at-least-once delivery and no persistence layer to operate.
- redis/redis — use instead when you already run Redis and need lightweight pub/sub or Streams alongside caching.
- eclipse-mosquitto/mosquitto — use instead when you only need an MQTT broker for IoT and don't need clustering or a broader messaging fabric.

## History

| Version | Date | Notes |
|---------|------|-------|
| Ruby gnatsd | 2011 | Original NATS as the Cloud Foundry message bus. |
| Go rewrite | 2015 | Server reimplemented in Go (`gnatsd`), later renamed `nats-server`. |
| 1.0 | 2018 | First 1.x stable line of the Go server. |
| 2.0 | 2019-06 | Accounts / multi-tenancy, nkeys + JWT auth, leaf nodes, gateways/superclusters[^3]. |
| 2.2 | 2021-03 | JetStream promoted to GA — persistence, KV/Object store foundation[^2]. |
| 2.9 | 2022 | JetStream improvements, source/mirror maturity. |
| 2.10 | 2023-09 | Subject-mapped transforms, compression, per-message TTL groundwork. |
| 2.11 | 2025 | Continued JetStream and clustering refinements[^4]. |

## References

[^1]: NATS project overview and history, nats.io. https://nats.io/about/
[^2]: Synadia, "JetStream is Generally Available" — NATS 2.2. https://nats.io/blog/nats-jetstream-preview-2-ga/
[^3]: NATS 2.0 release announcement (accounts, leaf nodes, superclusters). https://nats.io/blog/nats-server-2-release/
[^4]: nats-server releases. https://github.com/nats-io/nats-server/releases
[^5]: Trail of Bits / OSTIF security audit of NATS, April 2025. https://github.com/trailofbits/publications/blob/master/reviews/2025-04-ostif-nats-securityreview.pdf

## Tags

go, messaging, pub-sub, message-broker, jetstream, cloud-native, cncf, distributed-systems, edge, microservices, event-streaming, request-reply
