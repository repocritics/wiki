# IBM/sarama

> A pure-Go client library for Apache Kafka that speaks the wire protocol directly, with no cgo and no librdkafka dependency.

[GitHub repo](https://github.com/IBM/sarama) ·
[License: MIT](https://github.com/IBM/sarama/blob/main/LICENSE.md)

## Overview

Sarama is a Go client for Apache Kafka that implements the Kafka wire protocol natively in Go[^1]. Unlike the Confluent client, it does not wrap the C library `librdkafka`, so it builds and cross-compiles as ordinary Go code with no cgo toolchain. It has been in continuous development since 2013, which makes it one of the oldest and most-depended-on Kafka clients in the Go ecosystem — it is a transitive dependency of a large amount of infrastructure software (log shippers, stream processors, exporters).

The library was created and maintained at Shopify for most of its life; ownership and the canonical import path moved to IBM in 2023, and the module is now `github.com/IBM/sarama`[^2]. The old `github.com/Shopify/sarama` path still redirects on GitHub, but new code should import the IBM path, and this rename is the single most common reason older tutorials and Stack Overflow answers no longer compile cleanly.

Sarama's defining tension is coverage versus ergonomics. Because it reimplements the protocol by hand, it exposes a very large surface — a `Config` struct with dozens of fields, separate sync and async producers, and a consumer-group API that hands you the rebalance lifecycle to manage yourself. That control is the reason people reach for it, and also the reason it has a reputation for being easy to hold wrong.

## Getting Started

```bash
go get github.com/IBM/sarama
```

A minimal synchronous producer:

```go
package main

import (
	"log"

	"github.com/IBM/sarama"
)

func main() {
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true          // required for SyncProducer
	config.Producer.RequiredAcks = sarama.WaitForAll // acks=all

	producer, err := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
	if err != nil {
		log.Fatal(err)
	}
	defer producer.Close()

	partition, offset, err := producer.SendMessage(&sarama.ProducerMessage{
		Topic: "events",
		Value: sarama.StringEncoder("hello"),
	})
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("stored to partition %d at offset %d", partition, offset)
}
```

## Architecture / How It Works

Sarama is layered from raw protocol structs up to convenience APIs.

- **Broker / protocol layer.** Each Kafka request and response type (Produce, Fetch, Metadata, OffsetCommit, JoinGroup, SyncGroup, etc.) is a Go struct that knows how to encode and decode itself. `Client` maintains the cluster metadata — which broker leads which partition — and refreshes it on `RefreshMetadata` or on protocol errors that signal a stale view. This layer is why the correct `Config.Version` matters: Sarama only sends request versions it believes the broker supports, and defaults conservatively.

- **Producers.** `SyncProducer` blocks until the broker acknowledges; internally it is the `AsyncProducer` with a wait wrapped around it, which is why `Producer.Return.Successes` must be enabled for the sync path. `AsyncProducer` exposes `Input()`, `Successes()`, and `Errors()` channels — you must drain `Errors()` (and `Successes()` if `Return.Successes` is on) or the producer deadlocks once internal buffers fill. Batching, compression, and retry are configured on `Config.Producer`.

- **Consumer groups.** The `ConsumerGroup` interface runs the Kafka group-membership protocol (join/sync/heartbeat). You supply a `ConsumerGroupHandler` with three methods: `Setup`, `Cleanup`, and `ConsumeClaim`. `ConsumeClaim` receives a channel of messages for one assigned partition and is expected to loop until the channel closes — a close signals a rebalance, and returning promptly is what lets the group rebalance without hitting the session timeout. Offsets are marked with `MarkMessage` and committed on the configured interval; forgetting to mark, or blocking in `ConsumeClaim`, are the two classic bugs.

- **Partitioners and offset management** are pluggable. The default partitioner hashes the message key; manual, round-robin, and reference (Java-compatible) partitioners ship in the box.

The whole client is protocol-faithful rather than opinionated: it maps closely onto Kafka's own concepts instead of hiding them, which is a good fit if you already understand Kafka and a poor one if you wanted a queue abstraction.

## Production Notes

- **Drain the async channels.** The most common production hang is an `AsyncProducer` whose `Errors()` (or `Successes()`) channel is never read. Once the internal channel backs up, `Input() <- msg` blocks forever. Every async producer needs a goroutine consuming its return channels.

- **Set `Config.Version` explicitly.** The default is conservative and disables newer features (record headers, some consumer-group semantics, exactly-once building blocks). Pin it to your broker's version, e.g. `config.Version = sarama.V3_6_0_0`, or you will silently run against an old protocol.

- **Rebalance discipline.** `ConsumeClaim` must return quickly when its message channel closes. Long-running per-message work should be bounded or offloaded; otherwise a rebalance stalls, the session times out, and the group thrashes. This is the most-reported operational issue with Sarama consumer groups.

- **Idempotence and EOS are partial.** Sarama supports the idempotent producer and has transaction APIs, but its exactly-once story is less complete and less battle-tested than the JVM client or `librdkafka`. Teams needing strict EOS should validate carefully rather than assume parity.

- **Errors are values, and some are retriable.** Metadata staleness surfaces as typed errors (`ErrNotLeaderForPartition`, `ErrLeaderNotAvailable`); a metadata refresh usually resolves them, and Sarama retries internally per `Config` limits. Leaking these to the caller as fatal is a frequent mistake.

- **The mocks subpackage exists — use it.** `mocks` provides fake producers/consumers so tests do not require a live broker. For integration tests, a real or containerized Kafka is still the only way to catch protocol-version mismatches.

## When to Use / When Not

**Use when:**
- You want a pure-Go build with no cgo, easy cross-compilation, and no `librdkafka` at runtime.
- You need low-level control over the protocol, partitioning, and offset handling.
- You are maintaining existing Sarama code or the wider Go infra ecosystem that already depends on it.

**Avoid when:**
- You want a modern, smaller, more ergonomic API — `twmb/franz-go` is frequently preferred for new projects.
- You need the most complete exactly-once / transactional guarantees and want parity with the JVM client — the Confluent client is closer.
- You want a high-level queue abstraction rather than a faithful Kafka protocol client.

## Alternatives

- twmb/franz-go — pure-Go, newer design, generally cleaner consumer-group and transaction APIs; use it when starting a new Go+Kafka project without legacy Sarama code.
- confluentinc/confluent-kafka-go — cgo wrapper over librdkafka; use it when you want feature parity with the reference C client and can accept the cgo build requirement.
- segmentio/kafka-go — pure-Go with a `net`-style reader/writer API; use it when you prefer a simpler streaming interface over full protocol control.
- Shopify/sarama — the former home of this same project; use only to understand old imports — it now redirects to IBM/sarama.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-07 | First commit; created and maintained at Shopify[^1]. |
| 1.x | ongoing | Long-lived v1 line under semantic versioning; API stability per Go module rules. |
| transfer | 2023 | Ownership and canonical import path moved to `github.com/IBM/sarama`[^2]. |

As of mid-2026 the repository is actively maintained, with roughly 12,500 stars and 1,800 forks, recent pushes, and a low open-issue count relative to its size — signs of a mature project in steady maintenance rather than rapid feature growth.

## References

[^1]: Sarama README and repository, `github.com/IBM/sarama`. https://github.com/IBM/sarama
[^2]: Module import path `github.com/IBM/sarama` on the Go package index. https://pkg.go.dev/github.com/IBM/sarama

## Tags

go, kafka, kafka-client, messaging, streaming, event-streaming, producer-consumer, distributed-systems, protocol-client, library
