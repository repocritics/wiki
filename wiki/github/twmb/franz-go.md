# twmb/franz-go

> A feature-complete, pure-Go Apache Kafka client that tracks the Kafka protocol KIP-by-KIP.

[GitHub repo](https://github.com/twmb/franz-go) ·
[License: BSD-3-Clause](https://github.com/twmb/franz-go/blob/master/LICENSE)

## Overview

franz-go is a Go client library for Apache Kafka written and maintained largely
by a single author, Travis Bischel (twmb)[^1]. Its stated goal is to support
*every* client-facing Kafka feature from broker version 0.8.0 through the latest
releases (4.2+ as of 2026), and in practice it is the most protocol-complete Go
client: it implements EOS transactions, idempotent producing, eager and
cooperative group balancers, rack-aware fetching (KIP-881), share/queue-group
consuming (KIP-932), all compression codecs, and all SASL mechanisms. Any KIP
that only adds or changes wire messages is picked up for free via code
generation from the `pkg/kmsg` package[^2].

The library is pure Go with no cgo. This is the defining tradeoff against
`confluentinc/confluent-kafka-go`, which wraps the C library librdkafka: franz-go
cross-compiles trivially and needs no C toolchain, at the cost of not inheriting
librdkafka's decade of production hardening. It is also more opinionated and
lower-level than `IBM/sarama` — a large option surface and direct control over
polling, buffering, and commit semantics, which is power for experienced operators
and a footgun for newcomers. It is the default client in Redpanda's own tooling
(Redpanda Console), and Redpanda is a first-class supported broker target[^3].

## Getting Started

```bash
go get github.com/twmb/franz-go
```

```go
// One client both produces and consumes.
cl, err := kgo.NewClient(
	kgo.SeedBrokers("localhost:9092"),
	kgo.ConsumerGroup("my-group"),
	kgo.ConsumeTopics("foo"),
)
if err != nil {
	panic(err)
}
defer cl.Close()

ctx := context.Background()

// Produce (async; the callback reports the per-record result).
record := &kgo.Record{Topic: "foo", Value: []byte("bar")}
cl.Produce(ctx, record, func(_ *kgo.Record, err error) {
	if err != nil {
		fmt.Printf("produce error: %v\n", err)
	}
})

// Consume: poll in a loop, handle errors, iterate records.
for {
	fetches := cl.PollFetches(ctx)
	if errs := fetches.Errors(); len(errs) > 0 {
		panic(fmt.Sprint(errs)) // non-retriable errors surface here
	}
	for iter := fetches.RecordIter(); !iter.Done(); {
		fmt.Println(string(iter.Next().Value))
	}
}
```

## Architecture / How It Works

The repo is a **multi-module monorepo** with independently tagged packages, which
is central to how you depend on it:

- `pkg/kgo` — the client. Everything user-facing lives here.
- `pkg/kmsg` — the raw, code-generated protocol layer. Its own module with its own
  version line (e.g. `v1.x`); admin and low-level requests go through it.
- `pkg/kadm` — higher-level admin helpers (create/list topics, describe groups,
  offsets) built on top of the raw request path.
- `pkg/sr` — a schema-registry client plus a `Serde` type for encode/decode.
- `plugin/` — separately versioned metrics/logging adapters (`kprom`, `kzap`,
  `kgmetrics`, etc.), so pulling in Prometheus support does not drag its deps into
  every build.

The client's performance posture is deliberate: it "avoids channels and goroutines
where not necessary." Producing is buffered and batched per broker; consuming is
pull-based through `PollFetches`, which returns whatever has been fetched into
in-memory buffers. Requests default to each broker's maximum supported API version
(discovered via an ApiVersions request on connect), with `MaxVersions` to pin.
TLS and SASL are configured through dialer and option hooks rather than a config
struct, and observability is exposed through a neutral **hooks** interface
(connect/disconnect/read/write/throttle/batch events) that the plugin packages
translate into concrete metrics backends[^4].

Notably, the next-generation consumer group protocol (KIP-848) is implemented but
**deliberately hidden** — the maintainer flagged broker-side implementation
concerns and gated it behind an opt-in described in the 1.19.0 release notes[^5].
Shipping to spec but withholding what he considers unsafe is characteristic here.

## Production Notes

- **Pin `pkg/kmsg` alongside `pkg/kgo`.** Because they are separate modules,
  `go.mod` typically lists both (`franz-go v1.x` + `franz-go/pkg/kmsg v1.x`). A
  mismatched or floating kmsg can pull in protocol changes out of step with the
  client. Pin both explicitly.
- **You must poll continuously.** `PollFetches` is the only thing that drains
  fetch buffers and drives group heartbeats/rebalances. Slow or paused polling
  stalls the consumer and can trigger rebalances; blocking work inside the poll
  loop is the most common self-inflicted latency problem.
- **Autocommit can lose or duplicate records.** The default group autocommit marks
  everything returned by the last poll as committed, so a crash mid-processing can
  drop records. For at-least-once, commit explicitly (`CommitRecords`) after
  processing, and consider `BlockRebalanceOnPoll` so a rebalance cannot yank
  partitions out from under in-flight work.
- **Memory is buffer-driven.** `FetchMaxBytes`, `FetchMaxPartitionBytes`, and
  `MaxConcurrentFetches` govern RAM held for fetches; defaults are generous and
  high-throughput consumers should tune them down deliberately.
- **Idempotent producing is the default** and requires broker support
  (IDEMPOTENT_WRITE). Managed brokers differ: Microsoft Event Hubs rejects
  compressed produce, so you must set `ProducerBatchCompression(NoCompression)`[^3].
- **The API surface is large.** Correctness often depends on the right combination
  of variadic options; read the `docs/` directory (producing-and-consuming,
  transactions) before running EOS in production.

## When to Use / When Not

**Use when:**
- You want a pure-Go client with no cgo and painless cross-compilation.
- You need full Kafka feature coverage — transactions/EOS, cooperative rebalancing,
  share groups, rack-aware fetching — rather than a subset.
- You're on Redpanda or another Kafka-compatible broker and want a client that
  tracks the protocol closely.

**Avoid when:**
- You want a small, opinion-light API and are willing to trade features for it
  (`segmentio/kafka-go` is simpler).
- Your organization already standardizes on librdkafka semantics and tooling
  (`confluent-kafka-go` matches them exactly).
- You need the assurance of a large maintainer team; franz-go is effectively a
  single-maintainer project, which is a bus-factor consideration for critical
  infrastructure.

## Alternatives

- confluentinc/confluent-kafka-go — cgo wrapper over librdkafka; use it when you
  want librdkafka's battle-tested semantics and don't mind a C toolchain.
- segmentio/kafka-go — pure Go with a simpler reader/writer API; use it when you
  want minimal surface area and don't need full EOS/transaction support.
- IBM/sarama — the long-standing pure-Go client (formerly Shopify/sarama); use it
  when an existing codebase already depends on it, accepting its heavier API.
- redpanda-data/console — not a client but a consumer of franz-go; reference it to
  see franz-go used at scale.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-03-25 | Initial repository[^1]. |
| 0.x | 2020–2021 | Pre-1.0 development; protocol coverage and code-gen matured. |
| 1.0.0 | 2022 | First stable release; API stabilized under semver. |
| 1.x | 2022–2026 | Ongoing feature tracking: share groups (KIP-932), rack-aware fetch (KIP-881), KIP-848 gated. |
| 1.19.0+ | 2026 | Next-gen group protocol opt-in documented; current line ~1.21.x[^5]. |

## References

[^1]: franz-go repository and author (Travis Bischel, twmb). https://github.com/twmb/franz-go
[^2]: `pkg/kmsg` raw protocol / code-generation package. https://pkg.go.dev/github.com/twmb/franz-go/pkg/kmsg
[^3]: franz-go README — supported brokers and Event Hubs compression caveat. https://github.com/twmb/franz-go#works-with-any-kafka-compatible-brokers
[^4]: franz-go metrics & logging hooks and plugin packages. https://github.com/twmb/franz-go/tree/master/plugin
[^5]: franz-go README — next-generation group protocol hidden; opt-in per 1.19.0 release notes. https://github.com/twmb/franz-go/releases

## Tags

go, kafka, kafka-client, streaming, messaging, redpanda, event-streaming, distributed-systems, library
