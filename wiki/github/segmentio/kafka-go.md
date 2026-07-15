# segmentio/kafka-go

> A pure-Go Kafka client with no cgo dependency, exposing both a low-level connection API and standard-library-style Reader/Writer types.

[GitHub repo](https://github.com/segmentio/kafka-go) ·
[License: MIT](https://github.com/segmentio/kafka-go/blob/main/LICENSE)

## Overview

kafka-go is a Kafka client for Go written and open-sourced by Segment (acquired by Twilio in 2020)[^1]. It was built because the Go Kafka options available around 2017 did not fit Segment's needs: Shopify's `sarama` exposed the wire protocol at a low level and allocated heavily, while `confluent-kafka-go` wrapped the C library `librdkafka` via cgo, dragging a native dependency into every consumer[^2]. kafka-go's defining choice is to be pure Go — no cgo, no `librdkafka` — and to mirror standard-library idioms (`io`, `context`, deadlines) so the client feels like the rest of a Go program.

The library is deliberately two-layered. The `Conn` type is a thin wrapper over a single TCP connection to one broker and speaks the Kafka protocol directly; on top of it sit `Reader` and `Writer`, which handle reconnection, offset management, partition balancing, and batching. This layering is the source of both its ergonomics and its main tradeoff: because it is pure Go and prioritizes a clean API, it does not match the raw throughput of the librdkafka-backed client, and it has never implemented some newer protocol features (notably Kafka transactions / exactly-once semantics).

kafka-go suits teams that value a dependency-free build, context-aware cancellation, and code that reads like idiomatic Go, and who are producing and consuming at scales where librdkafka's extra throughput is not the deciding factor.

## Getting Started

```bash
go get github.com/segmentio/kafka-go
```

```go
package main

import (
	"context"
	"log"

	"github.com/segmentio/kafka-go"
)

func main() {
	// Consumer group Reader — commits offsets automatically on ReadMessage.
	r := kafka.NewReader(kafka.ReaderConfig{
		Brokers: []string{"localhost:9092"},
		GroupID: "consumer-group-id",
		Topic:   "topic-A",
		MaxBytes: 10e6, // 10MB max fetch
	})
	defer r.Close()

	for {
		m, err := r.ReadMessage(context.Background())
		if err != nil {
			break
		}
		log.Printf("%s/%d/%d: %s = %s", m.Topic, m.Partition, m.Offset, m.Key, m.Value)
	}
}
```

```go
// Writer — Addr + Balancer; the zero value is usable, no NewWriter needed.
w := &kafka.Writer{
	Addr:     kafka.TCP("localhost:9092"),
	Topic:    "topic-A",
	Balancer: &kafka.LeastBytes{},
}
err := w.WriteMessages(context.Background(),
	kafka.Message{Key: []byte("Key-A"), Value: []byte("Hello World!")},
)
```

## Architecture / How It Works

**`Conn`** is the primitive: one connection to one broker, exposing `WriteMessages`, `ReadBatch`, `CreateTopics`, `ReadPartitions`, and administrative calls. It sets read/write deadlines like a `net.Conn`. Everything higher level is built from it.

**`Reader`** consumes a single topic. In its simplest form it targets one `Partition`; set a `GroupID` and it becomes a consumer-group member with client-side rebalancing and broker-managed offsets. A number of methods change meaning under a group: `SetOffset` errors, and `Offset`, `Lag`, and per-partition `Stats` return `-1` because a group member does not own a fixed partition[^3]. `ReadMessage` auto-commits; `FetchMessage` + `CommitMessages` gives explicit control, and `CommitInterval` switches commits from synchronous to periodic.

**`Writer`** batches records per partition and flushes on `BatchSize`, `BatchTimeout`, or `Close`. Partition selection is pluggable via `Balancer`: `LeastBytes`, `RoundRobin`, `Hash`, `ReferenceHash`, `CRC32Balancer`, and `Murmur2Balancer`. The last three exist specifically to match the partitioning of sarama, librdkafka, and the canonical Java client respectively, so a Go producer can share a topic with clients in other languages without messages landing on different partitions[^4].

Since **v0.4**, the writer's connection pooling and protocol handling moved to a `Transport`, and a dedicated `protocol` package provides a more structured implementation of the Kafka API set. The older `kafka.NewWriter`/`WriterConfig` and the `Dialer` are deprecated in favor of `Writer` + `Transport`, though both still work[^5]. Compression codecs (gzip, snappy, lz4, zstd) live in subpackages; since v0.4 they auto-register, so consumers no longer need to blank-import them to decode compressed batches.

## Production Notes

- **No transactions / exactly-once.** kafka-go does not implement the idempotent or transactional producer, so it cannot give exactly-once semantics. Consumer-group offset commits are at-least-once; a crash after processing but before commit reproduces messages. If you need EOS, this client is the wrong choice.
- **Throughput ceiling.** Being pure Go, it generally does not reach the raw producer/consumer throughput of librdkafka-backed `confluent-kafka-go`. For most services this is irrelevant; for very high-volume pipelines, benchmark before committing.
- **`CommitInterval` trades safety for speed.** Periodic commits reduce broker round-trips but widen the window of reprocessed messages after a crash. Synchronous `CommitMessages` is the safe default.
- **Always `Close()` the Reader.** Without a graceful disconnect the broker keeps the member in the group until the session times out, delaying rebalancing for the next reader on that topic. Wire `Close()` into a `signal.Notify` handler — a bare `SIGINT`/`SIGTERM` will skip deferred closes.
- **TLS misconfiguration is opaque.** Connecting to a TLS-enabled cluster without setting `TLS` on the Conn/Reader/Writer surfaces as `io.ErrUnexpectedEOF`, not a clear handshake error.
- **One Reader = one topic.** To fan a non-group consumer across partitions you run multiple Readers. Multi-topic consumption in a single group is not the model; structure around it.
- **Auto-topic-creation default changed.** Implicit topic creation on write was the default only up to v0.4.30; newer code must set `AllowAutoTopicCreation: true` explicitly[^6].
- **Protocol/version coverage.** The README historically pins tested Kafka versions (0.10.1.0 through 2.7.1); newer broker features may not be implemented even if the connection succeeds[^7]. Verify support before relying on recent KIPs.

## When to Use / When Not

**Use when:**
- You want a pure-Go build with no cgo and no native `librdkafka` dependency.
- You value `context`-based cancellation and standard-library-shaped APIs.
- Your throughput needs are comfortably met and simplicity matters more than peak performance.
- You produce alongside sarama/Java/librdkafka clients and need matching partitioners.

**Avoid when:**
- You need Kafka transactions or exactly-once semantics.
- You need maximum producer/consumer throughput and can accept cgo.
- You depend on very recent broker protocol features not yet implemented.

## Alternatives

- IBM/sarama — the other major pure-Go client (formerly Shopify/sarama); lower-level and more allocation-heavy, but broader protocol coverage including transactions. Use when you need feature completeness in pure Go.
- confluentinc/confluent-kafka-go — cgo wrapper over librdkafka; highest throughput and full EOS support. Use when performance or exactly-once is the priority and cgo is acceptable.
- twmb/franz-go — modern pure-Go client with transactions, KRaft, and wide protocol coverage. Use when you want kafka-go-style ergonomics without the feature gaps.
- lovoo/goka — stream-processing framework (built on sarama) with state and processor abstractions. Use for stateful stream processing rather than raw client access.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-05 | Open-sourced by Segment; pure-Go, `Conn`-first design[^1]. |
| 0.3.x | ~2019 | `Reader`/`Writer`, consumer groups, explicit commits. |
| 0.4.0 | ~2020 | `Transport` + `protocol` package rewrite; `Dialer`/`WriterConfig` deprecated; compression codecs auto-register[^5]. |
| 0.4.30 | — | Last version to auto-create topics on write by default[^6]. |

## References

[^1]: kafka-go README, "Motivations" — Segment's rationale for a new pure-Go client. https://github.com/segmentio/kafka-go#motivations
[^2]: confluent-kafka-go is a cgo wrapper around librdkafka. https://github.com/confluentinc/confluent-kafka-go
[^3]: kafka-go README, "Consumer Groups" — method limitations when `GroupID` is set. https://github.com/segmentio/kafka-go#consumer-groups
[^4]: kafka-go README, "Compatibility with other clients" — Hash/ReferenceHash/CRC32/Murmur2 balancers. https://github.com/segmentio/kafka-go#compatibility-with-other-clients
[^5]: kafka-go README — `NewWriter`/`WriterConfig`/`Dialer` deprecation notes. https://github.com/segmentio/kafka-go#writer
[^6]: kafka-go README, "Writer" — auto topic creation was default up to v0.4.30. https://github.com/segmentio/kafka-go#writer
[^7]: kafka-go README, "Kafka versions" — tested 0.10.1.0 to 2.7.1. https://github.com/segmentio/kafka-go#kafka-versions

## Tags

go, golang, kafka, kafka-client, message-queue, event-streaming, producer, consumer, consumer-groups, pure-go, distributed-systems
