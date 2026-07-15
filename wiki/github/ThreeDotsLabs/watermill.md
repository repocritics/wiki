# ThreeDotsLabs/watermill

> A Go library for event-driven applications — pub/sub, CQRS, event sourcing and sagas over a single message-handler interface.

[GitHub repo](https://github.com/ThreeDotsLabs/watermill) ·
[Official website](https://watermill.io) ·
[License: MIT](https://github.com/ThreeDotsLabs/watermill/blob/master/LICENSE)

## Overview

Watermill is a messaging library built by Three Dots Labs, first published in 2018 and reaching a stable v1.0.0 that froze the public API against breaking changes without a major-version bump[^1]. Its thesis is that message-driven systems should feel as approachable as writing an HTTP handler: you receive a `Message`, you decide whether to emit new messages or return an error, and middleware handles the cross-cutting concerns (retries, poison queues, metrics, correlation IDs). It is a library, not a broker or a framework — it does not run a daemon, and it does not ship its own transport.

The defining design choice is that the transport is pluggable and lives outside the core module. The `watermill` module itself contains the `Message`, `Router`, middleware, the in-memory `GoChannel` Pub/Sub, and the CQRS layer; every real backend — Kafka, RabbitMQ, NATS Jetstream, Redis Streams, SQL (Postgres/MySQL), AWS SNS/SQS, Google Cloud Pub/Sub, and others — is a separate versioned repository under the same org[^2]. This keeps the core dependency-light and lets each adapter version independently, at the cost of a fragmented module graph and adapters that drift in maturity and feature coverage.

The audience is Go teams building asynchronous services who want a common handler/middleware abstraction across brokers rather than coding against each client SDK directly. The central tension is exactly that abstraction: Watermill's uniform `Publisher`/`Subscriber` interface is deliberately thin, so broker-specific semantics (partitioning, consumer groups, ack deadlines, ordering guarantees) leak through configuration and require you to understand the underlying system anyway.

## Getting Started

```bash
go get github.com/ThreeDotsLabs/watermill
# transports are separate modules, e.g.:
go get github.com/ThreeDotsLabs/watermill-kafka/v3
```

```go
package main

import (
	"context"

	"github.com/ThreeDotsLabs/watermill"
	"github.com/ThreeDotsLabs/watermill/message"
	"github.com/ThreeDotsLabs/watermill/pubsub/gochannel"
)

func main() {
	logger := watermill.NewStdLogger(false, false)
	pubSub := gochannel.NewGoChannel(gochannel.Config{}, logger)

	messages, err := pubSub.Subscribe(context.Background(), "example.topic")
	if err != nil {
		panic(err)
	}

	go func() {
		for msg := range messages {
			// do work...
			msg.Ack() // or msg.Nack() to redeliver
		}
	}()

	msg := message.NewMessage(watermill.NewUUID(), []byte("hello"))
	if err := pubSub.Publish("example.topic", msg); err != nil {
		panic(err)
	}
}
```

For real applications you typically use the `message.Router`, which wires subscribers to handler functions and applies middleware, rather than draining channels by hand.

## Architecture / How It Works

At the center is one handler signature[^3]:

```go
func(*message.Message) ([]*message.Message, error)
```

A `Message` carries a UUID, a `[]byte` payload, string `Metadata`, and an ack/nack channel. Everything else is built around moving that struct.

- **Publisher / Subscriber interfaces.** `Publisher.Publish(topic, ...*Message)` and `Subscriber.Subscribe(ctx, topic) (<-chan *Message, error)`. Any transport that satisfies these plugs in. Note the asymmetry with broker reality: a single topic string does not capture Kafka partitions or consumer groups, so adapters expose those through their own `Config` structs and marshalers.
- **Router.** The `message.Router` connects a subscriber topic to a handler and (optionally) a publisher topic for returned messages. It owns the ack lifecycle: a handler that returns without error acks the incoming message; a returned error triggers nack (and redelivery, if the transport supports it). The router also runs graceful shutdown and per-handler middleware.
- **Middleware.** Cross-cutting behavior is composed as decorators around handlers: `Retry`, `CircuitBreaker`, `Recoverer` (panic recovery), `PoisonQueue`, `Correlation`, `Throttle`, `Timeout`. This is where most operational behavior actually lives — the transport gives you delivery, middleware gives you resilience.
- **CQRS component.** A layer on top of the router providing `CommandBus`/`CommandProcessor` and `EventBus`/`EventProcessor`, with a pluggable `Marshaler` (JSON, Protobuf, Avro via generators). It is a convenience over the same handler model, not a separate runtime.
- **GoChannel.** An in-memory Pub/Sub used for tests, local development, and intra-process fan-out. It is the only transport in the core module and the fastest by a wide margin, because it never leaves the process.

The ack model is the part that repays careful reading: `Ack()`/`Nack()` are explicit and per-message, and their meaning depends on the transport. On some backends nack means immediate redelivery; on others it interacts with visibility timeouts or offset commits. Watermill standardizes the API surface but not the delivery semantics beneath it.

## Production Notes

- **The module graph is fragmented and independently versioned.** Core, each transport, and the CLI are separate Go modules with their own version numbers (e.g. `watermill-kafka/v3`, `watermill-sql/v4`, `watermill-nats/v2`). Upgrading core does not upgrade adapters; you must track compatibility per adapter, and a Watermill "upgrade" is really N coordinated upgrades. Read each adapter's release notes, not just the core changelog.
- **Adapter maturity varies.** The SQLite Pub/Sub is marked Beta in the README; others are mature and stress-tested. Do not assume feature parity or equal battle-testing across transports — the Kafka, AMQP, and SQL adapters see the most production use.
- **At-least-once by default; exactly-once is your problem.** Watermill delivers at-least-once on most transports, so handlers must be idempotent. The project ships a "exactly-once delivery counter" example precisely because you build that guarantee on top with deduplication or transactional outbox patterns — the library does not hand it to you.
- **Ordering is a transport property, not a library one.** GoChannel and single-partition setups preserve order; sharded/partitioned Kafka or concurrent handlers do not, unless you constrain partitioning and handler concurrency. The uniform interface can lull you into assuming ordering that the broker does not provide.
- **Benchmarks are order-of-magnitude, not SLAs.** The README's numbers (e.g. GoChannel ~315k publish/s vs RabbitMQ ~2.8k publish/s at 16-byte payloads, single 16-CPU VM) are explicitly rough estimates and swing widely with payload size, batching, and broker config[^4]. Treat the in-memory-vs-network gap as the real signal.
- **SQL Pub/Sub is a real option with real costs.** Using Postgres/MySQL as a queue (polling-based) is supported and convenient for transactional-outbox designs, but throughput is far below dedicated brokers and it puts queue load on your primary database. Good for correctness-critical, moderate-volume flows; not for high-fanout streaming.
- **Stability discipline is a genuine strength.** Each transport must pass a shared conformance suite, run 20x in parallel in stress mode, all under the race detector[^1]. This is a stronger correctness bar than most messaging libraries advertise.

## When to Use / When Not

**Use when:**
- You want one handler/middleware model across several brokers, or expect to swap transports.
- You're building CQRS / event-sourcing / saga-style flows in Go and want conventions rather than a bespoke framework.
- You want a transactional outbox via the SQL Pub/Sub, keeping messages in the same DB transaction as your writes.
- You value the tested, stress-verified conformance suite and a stable v1 API.

**Avoid when:**
- You only ever target one broker and are comfortable with its native client — the abstraction adds indirection for no portability gain.
- You need exactly-once or strict global ordering handed to you out of the box; Watermill gives you the tools, not the guarantee.
- You're not on Go, or you want a running broker/streaming engine rather than a library.
- You need every advanced feature of a specific broker; the uniform interface exposes the common subset, and edge features live in adapter config or not at all.

## Alternatives

- nats-io/nats.go — use the native NATS client when NATS is your only transport and you want its full feature set without a wrapper.
- segmentio/kafka-go — use directly when you are Kafka-only and prefer coding against Kafka semantics explicitly.
- confluentinc/confluent-kafka-go — use for the most complete librdkafka-backed Kafka feature coverage in Go.
- asynq (hibiken/asynq) — use for Redis-backed task/job queues with scheduling and a dashboard, rather than general event-driven messaging.
- Go standard channels — use when your "events" never leave the process; Watermill's GoChannel is thin sugar over this and a plain channel may be enough.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial commit | 2018-11 | Repository created; project introduced by Three Dots Labs[^5]. |
| v1.0.0 | 2019 | Stable release; public API frozen against breaking changes without a major bump[^1]. |
| Ongoing | 2019–2026 | Transports split into independently versioned modules (kafka v3, sql v4, nats v2, etc.); CQRS component and expanded adapter set. |

## References

[^1]: Watermill README — Stability section (v1.0.0 API stability, shared conformance tests run 20x in stress mode under `-race`). https://github.com/ThreeDotsLabs/watermill#stability
[^2]: Watermill README — Pub/Subs section, listing transports as separate `github.com/ThreeDotsLabs/watermill-*` modules. https://github.com/ThreeDotsLabs/watermill#pubsubs
[^3]: Watermill README — Background section, core handler interface `func(*Message) ([]*Message, error)`. https://github.com/ThreeDotsLabs/watermill#background
[^4]: Watermill README — Benchmarks section (single 16-CPU VM, 16-byte messages; described as rough estimates). https://github.com/ThreeDotsLabs/watermill#benchmarks
[^5]: "Introducing Watermill" — Three Dots Labs blog. https://threedots.tech/post/introducing-watermill/

## Tags

go, golang, event-driven, pub-sub, cqrs, event-sourcing, messaging, kafka, rabbitmq, nats, stream-processing, sagas
