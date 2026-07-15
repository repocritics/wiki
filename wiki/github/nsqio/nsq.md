# nsqio/nsq

> A decentralized realtime messaging platform that trades durability and ordering for operational simplicity and horizontal scale.

[GitHub repo](https://github.com/nsqio/nsq) ·
[Official website](https://nsq.io) ·
[License: MIT](https://github.com/nsqio/nsq/blob/master/LICENSE)

## Overview

NSQ is a distributed message queue written in Go, originally built at Bitly and
open-sourced in 2012[^1]. It targets high-throughput, at-least-once message
delivery for work distribution and event fan-out, and has historically been
cited as handling billions of messages per day in production[^2]. At ~25.7k
stars it is one of the better-known Go infrastructure projects, though its
release cadence has slowed considerably — the codebase is mature and stable
rather than actively evolving, with the last push in mid-2025.

The defining design decision is decentralization *by omission*: NSQ has no
broker replication, no consensus layer, and no ZooKeeper-style coordinator.
Each `nsqd` node owns its own queues on its own disk. This makes NSQ trivial to
deploy (single static binary, no runtime dependencies, all config on the command
line) and easy to scale horizontally by adding more nodes. The cost is paid on
two axes that operators must accept up front: messages are **not ordered**, and
messages queued on a node are **lost if that node's host or disk dies**. NSQ is
the right tool when those tradeoffs are acceptable and the wrong one when they
are not — most of the ecosystem's "NSQ hurt us" stories come from teams that
needed a durable, ordered log and reached for NSQ because it was easy to run.

## Getting Started

```bash
# macOS
brew install nsq
# or download static binaries for Linux/Darwin/FreeBSD/Windows from nsq.io
```

```bash
# 1. discovery daemon
nsqlookupd &

# 2. message daemon, pointed at lookupd
nsqd --lookupd-tcp-address=127.0.0.1:4160 &

# 3. admin web UI (optional)
nsqadmin --lookupd-http-address=127.0.0.1:4161 &

# publish a message over HTTP (topic auto-created)
curl -d 'hello world' 'http://127.0.0.1:4151/pub?topic=events'
```

```go
// consumer with the official go-nsq client
cfg := nsq.NewConfig()
consumer, _ := nsq.NewConsumer("events", "worker", cfg) // topic, channel
consumer.AddHandler(nsq.HandlerFunc(func(m *nsq.Message) error {
    log.Printf("got %s", m.Body)
    return nil // returning nil FINs the message; non-nil requeues it
}))
// discover nsqds via lookupd — do NOT hardcode nsqd addresses in consumers
consumer.ConnectToNSQLookupd("127.0.0.1:4161")
```

## Architecture / How It Works

NSQ is three cooperating daemons:

- **`nsqd`** — the workhorse. Receives, queues, and delivers messages. Each
  `nsqd` is fully independent and holds no knowledge of other `nsqd` instances.
- **`nsqlookupd`** — a directory/discovery service. `nsqd` nodes register the
  topics they host; consumers query `nsqlookupd` to learn *which* `nsqd`s carry
  a topic, then connect to all of them directly. It is eventually consistent and
  holds no message data — you run several for availability, and existing
  connections keep working even if every `nsqlookupd` is down.
- **`nsqadmin`** — a read-mostly web UI for inspecting topics, channels, and
  depth.

The core data model is **topics** and **channels**. A producer publishes to a
topic. Every channel created on that topic receives a *copy* of every message —
channels are the fan-out primitive. Within a single channel, multiple connected
consumers **load-balance** the messages. So "one channel = one logical consumer
group" and "N channels = N independent copies of the stream."

Delivery is **at-least-once**. Each in-flight message has a timeout
(`--msg-timeout`); if the consumer does not `FIN` it in time, or explicitly
`REQ`s it, the message is redelivered — so consumers must be idempotent.
Consumer flow control is explicit via the `RDY` state: a consumer advertises how
many messages it is ready to receive, giving NSQ backpressure without unbounded
buffering.

Persistence is a two-tier queue per topic/channel. Messages sit in an in-memory
queue up to `--mem-queue-size`; overflow spills to a disk-backed queue
(`diskqueue`). The disk queue is **not** fsync'd per message by default — it
flushes on `--sync-every` count or `--sync-timeout` interval — so a crash can
lose the most recently written messages. Setting `--mem-queue-size=0` forces
everything through disk for stronger durability at a throughput cost.

Clients talk a simple length-prefixed **TCP protocol**[^3] for
subscribe/publish; `nsqd` also exposes an **HTTP/HTTPS** endpoint for publishing
and admin. TLS, Snappy, and Deflate are negotiated per-connection. NSQ is
payload-agnostic — bodies are opaque bytes (JSON, MsgPack, Protobuf, anything).

## Production Notes

- **No replication, single-node ownership.** A message lives on exactly one
  `nsqd`. Lose that host and you lose its queued (and un-synced) messages. There
  is no built-in mechanism to replicate or fail over queued data. Plan capacity
  and durability around per-node loss.
- **Colocate `nsqd` with producers.** The canonical topology runs an `nsqd` on
  (or next to) each producing host, so a network partition can't stop local
  publishing. Consumers, by contrast, connect out to *many* `nsqd`s via lookupd.
- **`RDY` starvation with many nodes.** A consumer connected to a large number
  of `nsqd`s with a low `max-in-flight` can distribute its RDY count so thinly
  that some connections get zero and their messages sit undelivered. Keep
  `max-in-flight` ≥ the number of `nsqd`s a consumer connects to.
- **Requeue storms.** A too-short `--msg-timeout` under slow handlers causes
  messages to time out and requeue while still being processed, multiplying
  load and duplicates. Tune timeout to real handler latency.
- **Message size cap.** Default max message size is ~1 MB (`--max-msg-size`);
  large payloads are an anti-pattern — put a pointer in the message and the blob
  elsewhere.
- **No auth by default.** `nsqd` supports optional TLS and an external HTTP auth
  server (`--auth-http-address`), but out of the box the endpoints are open.
  Don't expose `nsqd`/`nsqlookupd` ports on untrusted networks.
- **Ephemeral topics/channels** (`#ephemeral` suffix) never touch disk and are
  discarded when no consumer is connected — useful for lossy telemetry, wrong
  for anything you need to keep.
- **`master` is the dev branch** and the README warns it may be unstable; pin to
  a tagged release for production.

## When to Use / When Not

**Use when:**
- You need high-throughput, at-least-once work distribution or fan-out and can
  tolerate duplicates and occasional loss on host failure.
- You want dead-simple operations: static binaries, no ZooKeeper/etcd, no
  broker cluster to babysit.
- Your topology is many independent producers and elastic consumer pools.

**Avoid when:**
- You need strict message ordering — NSQ does not provide it.
- You need durability guarantees against node loss, or exactly-once semantics.
- You need long retention, replay, or log/stream semantics (consumer offsets,
  time-travel) — NSQ deletes messages once delivered.
- You need rich routing (topic exchanges, priorities, AMQP) or per-message TTL
  policies.

## Alternatives

- apache/kafka — use instead when you need a durable, ordered, replayable
  partitioned log with long retention, at the cost of heavier operations.
- rabbitmq/rabbitmq-server — use when you need flexible AMQP routing, priorities,
  and mirrored/quorum durability rather than raw fan-out throughput.
- nats-io/nats-server — use when you want comparably lightweight messaging but
  with an optional durable stream layer (JetStream) and clustering.
- redis/redis — use Redis Streams for lightweight at-least-once streaming when
  you already run Redis and don't want another system.
- apache/pulsar — use when you need Kafka-like durability plus multi-tenancy and
  tiered storage, and can absorb the operational complexity.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-sourced | 2012 | Released by Bitly; realtime messaging at scale[^1]. |
| v1.0.0 | 2016 | First 1.0; stabilized protocol and deployment story. |
| v1.1.0 | 2018 | Go module support, protocol/TLS improvements. |
| v1.2.x | 2019–2020 | Maintenance releases, diskqueue and client fixes. |
| v1.3.0 | 2022 | Latest feature release line; bug fixes and Go version bumps. |

(Exact per-release dates were not verified against a live release API for this
page — see Issues. The GitHub repository was created 2012-05-12 and last pushed
2025-07-13.)

## References

[^1]: NSQ project README and site — "a realtime distributed messaging platform designed to operate at scale, handling billions of messages per day." https://nsq.io
[^2]: NSQ — Features & Guarantees. https://nsq.io/overview/features_and_guarantees.html
[^3]: NSQ TCP protocol specification. https://nsq.io/clients/tcp_protocol_spec.html

## Tags

go, message-queue, messaging, distributed-systems, pub-sub, at-least-once, realtime, queue, event-driven, infrastructure
