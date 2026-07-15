# nats-io/nats.go

> The canonical Go client for NATS — a fire-and-forget pub/sub core with an opt-in persistence layer (JetStream) bolted on top.

[GitHub repo](https://github.com/nats-io/nats.go) ·
[Official website](https://nats.io) ·
[License: Apache-2.0](https://github.com/nats-io/nats.go/blob/main/LICENSE)

## Overview

`nats.go` is the reference client for NATS, a CNCF-hosted messaging system maintained by Synadia. The repository dates to 2012[^1] (originally under Apcera, later renamed from `go-nats` to `nats.go`), which makes it one of the older continuously-maintained Go networking libraries. It is the client every other NATS client is measured against, and the one the server team develops against.

The defining tension is that NATS is really two systems sharing one connection. **Core NATS** is at-most-once, in-memory, fire-and-forget pub/sub: a publish that reaches no subscriber, or a subscriber that falls behind, silently drops messages. There is no ack, no persistence, no replay. **JetStream** is a separate persistence layer (streams, durable consumers, key/value, object store) added years later that provides at-least-once delivery, retention, and replay. Both are reached through the same `nats.Conn`, and a large share of production incidents come from developers assuming core NATS gives them queue-like durability guarantees it never promised.

The client is small in surface area (connect, publish, subscribe, request) but the operational semantics — buffering, flushing, slow-consumer limits, reconnect behavior — are where the real complexity lives.

## Getting Started

```bash
go get github.com/nats-io/nats.go@latest
```

Note the module path stays `nats.go` (no `/v2`) even at v1.50+; only the server module is versioned `nats-server/v2`[^2].

```go
package main

import (
	"fmt"
	"time"

	"github.com/nats-io/nats.go"
)

func main() {
	nc, err := nats.Connect(nats.DefaultURL) // nats://127.0.0.1:4222
	if err != nil {
		panic(err)
	}
	defer nc.Drain() // preferred over Close(): flushes and unsubscribes cleanly

	nc.Subscribe("greet.*", func(m *nats.Msg) {
		m.Respond([]byte("hello " + m.Subject))
	})

	// Request-reply: publishes to an inbox and waits for one answer.
	msg, err := nc.Request("greet.world", nil, time.Second)
	if err != nil {
		panic(err) // nats.ErrNoResponders if nothing is subscribed
	}
	fmt.Println(string(msg.Data))
}
```

## Architecture / How It Works

A `nats.Conn` is a single multiplexed TCP connection. All subscriptions on that connection share it; the client demultiplexes incoming frames by subscription ID. Subjects are dot-delimited tokens with two wildcards: `*` matches one token, `>` matches the tail (`foo.>`).

**Delivery model.** Publishing is asynchronous and buffered — `Publish` writes to an outbound buffer and returns `nil` immediately; it does **not** mean the server received the message. `Flush` (or `FlushTimeout`) round-trips a PING/PONG to confirm the buffer drained. Async subscriptions (`Subscribe`) dispatch each message to your callback from a per-subscription goroutine, so a blocking callback stalls only that subscription's queue. `SubscribeSync` and `ChanSubscribe` hand delivery back to you.

**Slow consumers.** Each subscription has pending limits (default 512k messages / 64 MB). When a callback can't keep up, the client raises `ErrSlowConsumer` via the async error handler and drops messages for that subscription rather than blocking the whole connection. This is a frequent surprise: the drop is local to the client, silent unless you register `nats.ErrorHandler`.

**Reconnection.** The client maintains a server pool (seeded URLs plus servers auto-discovered from the cluster gossip), and on disconnect it reconnects transparently. Messages published while disconnected go into a reconnect buffer (default 8 MB); if it overflows, publishes error out. `DisconnectErrHandler` / `ReconnectHandler` / `ClosedHandler` observe the lifecycle.

**JetStream.** There are two client APIs. The older `nc.JetStream()` context API and the newer standalone `jetstream` package (`jetstream.New(nc)`), which is the recommended one; the legacy API is documented as superseded[^3]. JetStream turns durable state into stream/consumer objects with explicit `Ack()`, pull/push consumers, and `Consume` callbacks. `micro` is a separate services framework (request/reply with discovery and stats), released as beta[^4].

**Auth.** Username/password, token, TLS client certs, and the NATS-native decentralized scheme (NKeys + signed JWTs via `UserCredentials`). The private key never enters the core library — signing happens in a callback, and the helper wipes key material from memory between reconnects.

## Production Notes

- **`Publish` success is not delivery.** For core NATS there is no delivery receipt at all; the message is gone whether or not anyone heard it. If you need durability or acks, you need JetStream — this is the single most common design mistake.
- **Flush before you exit.** A program that publishes and immediately returns can lose buffered messages. Use `Drain()` (async, flushes then closes) or `Flush()` before shutdown. `Drain` is strongly preferred for responders so in-flight requests complete.
- **Max payload is server-enforced.** Default 1 MB. The client will reject oversized publishes with `ErrMaxPayload`; raising it is a server config change, and large messages defeat NATS's latency profile — use JetStream object store or chunking instead.
- **Slow-consumer drops are silent by default.** Always register `nats.ErrorHandler`; without it, `ErrSlowConsumer` message loss is invisible. Tune `SetPendingLimits` for high-fan-in subscriptions.
- **JetStream is at-least-once, not exactly-once.** Redelivery happens on missed acks; consumers must be idempotent. Double-ack (`AckSync`) trades latency for confirmation. Pull consumers scale better than push for most workloads.
- **Reconnect buffer can drop.** During long partitions, publishes beyond `ReconnectBufSize` fail. Applications that must not lose data during outages should publish through JetStream (which persists server-side) rather than relying on core reconnect buffering.
- **Goroutine-per-subscription callbacks.** Heavy work inside a `Subscribe` callback blocks that subscription's delivery. Offload to a worker pool, or use `ChanSubscribe` with your own concurrency.
- **API migration.** Codebases on the legacy `nc.JetStream()` API will keep working but new features land in the `jetstream` package; plan the migration rather than mixing both.

## When to Use / When Not

**Use when:**
- You want low-latency pub/sub, request/reply, or work-queue fan-out with a tiny operational footprint (one small server binary, no ZooKeeper/broker cluster to babysit).
- You need both ephemeral messaging and optional persistence (JetStream) from the same client and connection.
- You're building cloud-native microservices and want subject-based addressing, queue groups, and built-in service discovery (`micro`).

**Avoid when:**
- You need Kafka-style long-term log retention with a mature stream-processing ecosystem (Flink, ksqlDB) — JetStream covers replay but not that ecosystem.
- You need complex broker-side routing topologies (exchanges, dead-letter routing) that AMQP/RabbitMQ models directly.
- Your team will treat core NATS as a durable queue without adopting JetStream — the semantics mismatch will bite.

## Alternatives

- nats-io/nats.py, nats-io/nats.rs — official NATS clients for other languages; use when your service isn't Go, not as a Go alternative.
- confluentinc/confluent-kafka-go — use when you need Kafka's durable partitioned log, consumer-group replay, and stream-processing ecosystem.
- rabbitmq/amqp091-go — use when you need AMQP's rich broker-side routing (exchanges, bindings, dead-letter queues).
- redis/go-redis — use when you're already on Redis and pub/sub or Streams needs are simple and co-located with your cache.
- apache/pulsar-client-go — use when you need multi-tenant, geo-replicated tiered-storage streaming out of the box.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-08 | First Go client, under Apcera[^1]. |
| rename | ~2019 | Repo/module renamed `go-nats` → `nats.go` (`github.com/nats-io/nats.go`)[^2]. |
| JetStream | ~2021 | JetStream client support lands (NATS Server 2.2 era)[^3]. |
| micro | ~2023 | `micro` services API released as beta[^4]. |
| jetstream pkg | ~2023 | New standalone `jetstream` package supersedes legacy JetStream context[^3]. |
| v1.5x | 2026 | Ongoing v1 line; API-stable, expands structs/interfaces without breaking changes[^5]. |

## References

[^1]: `nats-io/nats.go` repository, created 2012-08-15. https://github.com/nats-io/nats.go
[^2]: Installation and module path notes, `nats.go` README. https://github.com/nats-io/nats.go#installation
[^3]: JetStream client documentation; new `jetstream` package supersedes the legacy API. https://github.com/nats-io/nats.go/blob/main/jetstream/README.md
[^4]: `micro` services API (beta). https://github.com/nats-io/nats.go/blob/main/micro/README.md
[^5]: Backward compatibility policy (adding fields/methods is not breaking; supports the two latest minor Go versions), `nats.go` README. https://github.com/nats-io/nats.go#backward-compatibility

## Tags

go, golang, messaging, pub-sub, jetstream, nats, cloud-native, microservices, streaming, message-queue, event-driven
