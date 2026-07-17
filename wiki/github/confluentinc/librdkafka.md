# confluentinc/librdkafka

> The C/C++ implementation of the Apache Kafka protocol — and the shared engine underneath the Python, Go, .NET, and most other non-JVM Kafka clients.

[GitHub repo](https://github.com/confluentinc/librdkafka) ·
[License: BSD-2-Clause](https://github.com/confluentinc/librdkafka/blob/master/LICENSE)

## Overview

librdkafka is a C library that speaks the Apache Kafka wire protocol, exposing Producer, Consumer, and Admin clients plus a thin C++ wrapper. It was started by Magnus Edenhill in 2012 and maintained largely solo for years; Edenhill joined Confluent, and Confluent Inc. took over copyright and stewardship in 2023[^1]. It is not a JVM project and shares no code with the canonical Java client — it is an independent reimplementation of the same protocol.

Its outsized importance comes from being a *substrate*, not just one client among many. The official Confluent clients for Python (confluent-kafka-python), Go (confluent-kafka-go), and .NET (confluent-kafka-dotnet) are bindings over this exact library, as are community bindings for Rust, PHP, Ruby, Node.js, and others[^2]. A bug fixed here fixes it for a large share of the non-Java Kafka ecosystem; a config property added here surfaces across all of them. The README advertises figures exceeding 1M msgs/sec producer and 3M msgs/sec consumer[^3], which is plausible for batched, compressed, in-datacenter workloads and misleading as a single-message-latency expectation.

The defining tension is that librdkafka is a low-level, poll-driven C library asked to serve high-level, garbage-collected languages. Most operational surprises trace back to that impedance mismatch: callbacks that only fire when you poll, queues that must be drained or flushed by hand, and configuration that mirrors the Java client's names but not always its defaults.

## Getting Started

```bash
# macOS
brew install librdkafka
# Debian/Ubuntu (Confluent APT repo) — package: librdkafka-dev
apt install librdkafka-dev
# or build from source:
./configure && make && sudo make install
```

```c
// producer.c — link with: cc producer.c -lrdkafka
#include <librdkafka/rdkafka.h>

int main(void) {
    char errstr[512];
    rd_kafka_conf_t *conf = rd_kafka_conf_new();
    rd_kafka_conf_set(conf, "bootstrap.servers", "localhost:9092",
                      errstr, sizeof(errstr));

    rd_kafka_t *rk = rd_kafka_new(RD_KAFKA_PRODUCER, conf,
                                  errstr, sizeof(errstr));

    rd_kafka_producev(rk,
        RD_KAFKA_V_TOPIC("events"),
        RD_KAFKA_V_VALUE("hello", 5),
        RD_KAFKA_V_END);

    rd_kafka_poll(rk, 0);        // serve delivery-report callbacks
    rd_kafka_flush(rk, 10000);   // block until queue drains — REQUIRED
    rd_kafka_destroy(rk);
    return 0;
}
```

The `flush` before `destroy` is not optional. `producev` only enqueues; without a flush (or enough polling) buffered messages are dropped on shutdown.

## Architecture / How It Works

librdkafka is asynchronous and thread-per-broker internally. Creating a client handle (`rd_kafka_t`) spins up a background thread per connected broker plus internal maintenance threads. Application calls do not talk to the network directly — they move messages through internal queues that the broker threads drain.

- **Producer path.** `rd_kafka_produce`/`producev` appends to a per-partition queue, is non-blocking, and returns immediately. Messages accumulate until `linger.ms` (`queue.buffering.max.ms`) elapses or `batch.num.messages` is reached, then are batched, optionally compressed, and sent. Delivery outcomes come back through a delivery-report callback that only fires while the application calls `rd_kafka_poll()`. If you never poll, the report queue fills and `produce` starts returning `RD_KAFKA_RESP_ERR__QUEUE_FULL`.
- **Consumer path.** The high-level `KafkaConsumer` (via `rd_kafka_consumer_poll`) handles group membership, rebalancing, and offset commits. A single poll call drives fetching, heartbeats, and rebalance callbacks; stop polling and the consumer is silently evicted from its group.
- **Configuration.** Everything is string key/value pairs on `rd_kafka_conf_t`, deliberately mirroring the Java client's property names (`acks`, `compression.type`, `enable.idempotence`, `max.in.flight`). Names match; some historical defaults did not.
- **Idempotence/EOS.** `enable.idempotence=true` transparently sets `acks=all`, bounds in-flight requests, and enables retries with sequence numbers. Transactions build on top via the transactional producer API.
- **Observability.** A statistics callback emits a large JSON blob (`statistics.interval.ms`) describing per-broker/per-partition queue depths, RTT, and lag[^4] — the primary window into what the background threads are doing.

Because bindings are thin FFI layers, this queue-and-poll model leaks upward: the poll requirement in Python/Go/.NET exists because it exists here.

## Production Notes

- **You must poll, or things break quietly.** The most common librdkafka bug in every binding is a producer that never calls `poll()` — delivery callbacks never fire, memory grows, and eventually `produce` fails with queue-full. Treat `poll()` as a mandatory heartbeat, not an optional callback pump.
- **Flush on shutdown or lose data.** `rd_kafka_flush()` with a real timeout before `destroy` is required for at-least-once delivery. This is the single most frequent cause of "Kafka is dropping my messages" reports.
- **Backpressure is config, not automatic.** `queue.buffering.max.messages` / `queue.buffering.max.kbytes` bound the internal buffer. A slow broker plus a fast producer fills these; decide whether `produce` should block or error (`RD_KAFKA_CONF_MAX_MSG_SIZE`, block-on-full behavior) before it happens in production.
- **Linking is a recurring tax.** OpenSSL, Cyrus SASL/Kerberos (GSSAPI), and zstd are optional dependencies resolved at build time. Static builds, cross-compilation, and SASL/GSSAPI on non-Linux platforms are where most build failures land. Static zstd linking needs zstd >= 1.2.1 to embed the original size for faster consumer decompression[^3].
- **Broker-version handshake.** Against very old brokers, `api.version.request` may need to be disabled with an explicit `broker.version.fallback`. Modern brokers negotiate automatically.
- **Config defaults drift from the Java client.** Do not assume identical defaults across the two clients; verify safety-critical properties (`acks`, `enable.idempotence`, `retries`) explicitly rather than trusting the name match.
- **Version pinning matters at the binding layer.** Because a binding embeds a specific librdkafka version, upgrading the Python/Go/.NET client to fix a protocol bug can change producer/consumer behavior; read librdkafka's own changelog, not just the wrapper's.
- **Share consumers (KIP-932, "Queues for Kafka") are Preview only**, C API only, and require broker >= 4.2 — not for production reliance yet[^5].

## When to Use / When Not

**Use when:**
- You are on a non-JVM stack (C, C++, Rust, Python, Go, .NET) and need a Kafka client — you are almost certainly using it already through a binding.
- You need EOS, idempotent, or transactional producers outside the JVM.
- You want the reference non-Java protocol implementation that tracks new KIPs closely.

**Avoid when:**
- You are on the JVM: use the official Apache Kafka Java client, which is the protocol's canonical implementation.
- You want a high-level, batteries-included API with no manual poll/flush lifecycle — reach for a binding with framework integration rather than raw C.
- You need a pure-language implementation with no native build step (some teams prefer a from-scratch Go/Rust client to avoid cgo/FFI and the linking tax).

## Alternatives

- apache/kafka — the canonical Java/Scala client; use it instead when you are on the JVM.
- twmb/franz-go — pure-Go Kafka client; use it when you want to avoid cgo and the librdkafka linking story in Go.
- segmentio/kafka-go — pure-Go client with a simpler API; use it for straightforward Go workloads without EOS-heavy needs.
- IBM/sarama — long-standing pure-Go client; use it where its ecosystem and middleware already exist, accepting its rougher edges.
- edenhill/kcat — command-line producer/consumer built on librdkafka (formerly kafkacat); use it for shell-level testing and debugging, not as a library.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial commit | 2012-09-19 | Started by Magnus Edenhill[^1]. |
| 0.11.0 | 2017 | Idempotent producer, message headers, EOS groundwork. |
| 1.0.0 | 2019 | Stable C API/ABI guarantee, transactions/EOS, sticky partitioner. |
| — | 2023 | Confluent Inc. assumes copyright and stewardship[^1]. |
| 2.x | 2023–2026 | KIP tracking, OAUTHBEARER OIDC, cooperative rebalancing maturity, KIP-932 share-consumer Preview[^5]. |

Exact minor-version dates are omitted where not verified against the release list; consult the GitHub releases page for precise tags[^6].

## References

[^1]: License and copyright header, librdkafka LICENSE — "Copyright (c) 2012-2022 Magnus Edenhill, 2023 Confluent Inc." https://github.com/confluentinc/librdkafka/blob/master/LICENSE
[^2]: "Language bindings" section, librdkafka README — bindings for Python, Go, .NET, Rust, PHP, Ruby, Node.js, and others. https://github.com/confluentinc/librdkafka#language-bindings
[^3]: librdkafka README — throughput figures and static-zstd linking note. https://github.com/confluentinc/librdkafka/blob/master/README.md
[^4]: Statistics metrics reference. https://github.com/confluentinc/librdkafka/blob/master/STATISTICS.md
[^5]: KIP-932 "Queues for Kafka" share-consumer support, marked Preview and C-API-only in the README feature list. https://github.com/confluentinc/librdkafka/blob/master/INTRODUCTION.md
[^6]: librdkafka releases. https://github.com/confluentinc/librdkafka/releases

## Tags

c, cpp, apache-kafka, kafka-client, messaging, streaming, producer-consumer, native-library, high-throughput, distributed-systems
