# edenhill/kcat

> A netcat for Apache Kafka: a single small C binary that produces to and consumes from a cluster from the shell.

[GitHub repo](https://github.com/edenhill/kcat) ·
[License: BSD-2-Clause](https://github.com/edenhill/kcat/blob/master/LICENSE)

## Overview

kcat is a generic, non-JVM command-line producer and consumer for Apache Kafka (broker >=0.8), written in C on top of librdkafka[^1]. The tag line in its own README — "think of it as a netcat for Kafka" — is the accurate mental model: you pipe stdin into a topic, or a topic into stdout, and compose it with the rest of a Unix pipeline. Statically linked, the binary is around 150 KB, which is what makes it the default reach-for tool on servers and in containers where dragging in the JVM and the official `kafka-console-*` scripts is not worth it.

The project was called **kafkacat** for most of its life; it was renamed to kcat in August 2021 to comply with the Apache Software Foundation's trademark policy on the "kafka" name[^2]. Nothing else changed at the rename, so a great deal of external documentation, package names (`apt-get install kafkacat`), and blog posts still say kafkacat and remain correct. Both names refer to the same tool and the same author, Magnus Edenhill.

The defining tension of kcat is that it is a thin shell over librdkafka. That is its strength — it inherits a mature, protocol-complete Kafka client and every `-X property=value` maps directly onto a librdkafka config knob — and its ceiling: kcat is essentially feature-complete and lightly maintained. Its last tagged release is 1.7.0 (August 2021), with the widely-used Docker image pinned at 1.7.1, and commit activity since has been sparse[^3]. This is a stable-and-quiet tool, not an actively-evolving one.

## Getting Started

```bash
# macOS
brew install kcat
# Debian/Ubuntu (still packaged under the old name)
apt-get install kafkacat
# or run the published image
docker run -it --network=host edenhill/kcat:1.7.1 -b BROKER -L
```

```bash
# Produce: pipe stdin, newline-delimited, into a topic
echo "hello kafka" | kcat -b mybroker -t mytopic -P

# Consume: read a topic to stdout, then exit at end-of-partition
kcat -b mybroker -t mytopic -C -e

# High-level balanced consumer group across two topics
kcat -b mybroker -G mygroup topic1 topic2

# Metadata listing — brokers, topics, partitions, ISRs
kcat -b mybroker -L
```

The four modes are selected by a single flag: `-P` produce, `-C` consume, `-G` consumer group, `-L` metadata list (`-Q` for offset-by-timestamp queries, `-M` for the in-memory mock cluster). Absent an explicit `-P`/`-C`, kcat infers mode from whether stdin is a terminal.

## Architecture / How It Works

kcat is a thin front-end; the actual Kafka protocol work lives in **librdkafka**, the C/C++ Kafka client also authored by Edenhill (now maintained under confluentinc/librdkafka)[^4]. Almost every operational behavior — batching, compression, retries, idempotence, SASL/SSL, broker version negotiation — is a librdkafka feature that kcat merely exposes. This is why the man page directs you to librdkafka's `CONFIGURATION.md` for the full `-X` surface, and why kcat gains protocol features essentially for free when linked against a newer librdkafka.

The output side is a small format engine. `-f 'Topic %t[%p], offset: %o, key: %k, payload: %s\n'` interpolates message fields into an arbitrary template, and `-J` emits a JSON envelope per message (topic, partition, offset, key, headers, payload). The input side is a delimiter splitter: `-D` sets the message delimiter, `-K` the key/value delimiter, `-Z` treats empty values as NULL (tombstones for compacted topics).

Optional deserialization is layered on top. With libyajl compiled in, JSON support is available; with libserdes and libavro-c, `-s avro -r http://schema-registry` decodes Confluent Schema-Registry Avro to JSON. There is also a generic pack-style deserializer (`-s value='i$'`, `-s key='hB s'`) for fixed binary layouts. All of these are compile-time optional, which is why a distro package and a hand-built binary can differ in capability.

Two features go beyond "netcat": transactional/idempotent production (`-X transactional.id=...`, `-X enable.idempotence=true`, again just librdkafka), and `-M`, an ephemeral in-memory mock Kafka cluster (backed by librdkafka's mock broker) suitable for local testing and, because it does no disk I/O, for quick throughput experiments.

## Production Notes

- **Capability depends on the build.** JSON and Avro are optional at compile time. A statically-linked or distro binary may lack `-J`, `-s avro`, or schema-registry support entirely; the `bootstrap.sh` script exists precisely to fetch and statically link librdkafka + yajl for a self-contained build. When a flag "does nothing," suspect the build before the cluster.
- **librdkafka version is the real dependency.** Broker compatibility, SASL mechanisms (OAUTHBEARER, SCRAM), and default behaviors are set by the linked librdkafka, not by kcat's own version. An old system librdkafka behind a new broker is the most common source of authentication and negotiation surprises; pin or bundle it deliberately.
- **`-o` offset semantics are a footgun.** `-o -2000` means "last 2000 messages," `-o beginning`/`-o end`, and `-o s@<ms>`/`-o e@<ms>` select by timestamp. Consuming without `-e` blocks and follows the topic indefinitely — fine interactively, a hang in scripts. Pair `-C` with `-e` in any automated pipeline.
- **Docker networking is the usual failure.** With `edenhill/kcat` in a container connecting to Dockerized brokers, the broker's advertised listeners (not kcat) determine reachability; `--network=host` or matching the compose network is required. This is a Kafka listener/advertised-address problem, not a kcat bug.
- **Maintenance is minimal.** No release since 1.7.0 (2021) and low commit volume mean bugs and feature requests can sit; the tool is treated as done. For new work you are effectively adopting a frozen tool that stays useful only because librdkafka underneath keeps up.
- **SASL/SSL is all `-X` passthrough.** e.g. `-X security.protocol=SASL_SSL -X sasl.mechanism=PLAIN -X sasl.username=… -X sasl.password=…`. Credentials on the command line land in shell history and `ps`; prefer a `-F config-file` or `~/.config/kcat.conf`.

## When to Use / When Not

**Use when:**
- You need to produce, consume, or inspect a cluster from a shell or a script without a JVM.
- You want a tiny, pipeable binary for containers, debugging, smoke tests, or one-off data movement.
- You need quick metadata (`-L`), offset-by-timestamp lookups (`-Q`), or ad-hoc Avro/JSON decoding at the terminal.

**Avoid when:**
- You want a supported, actively-developed CLI — kcat is stable but effectively frozen.
- You need a rich TUI, cluster management (ACLs, configs, reassignment), or connector/registry administration — kcat is data-plane only.
- You are building a production application rather than operating — link librdkafka (or a JVM/Go/Python client) directly instead of shelling out.

## Alternatives

- confluentinc/librdkafka — the C library kcat wraps; link it directly when embedding Kafka in an app rather than scripting the shell.
- apache/kafka — ships `kafka-console-producer.sh` / `kafka-console-consumer.sh`; use these when you already have the JVM tooling and want the reference implementation.
- redpanda-data/redpanda — its `rpk` CLI is a modern, actively-maintained alternative for produce/consume plus full cluster admin, especially against Redpanda but also Kafka.
- deviceinsight/kafkactl — a Go CLI with config contexts and admin operations; use when you want YAML-driven multi-cluster management, not just piping.
- birdayz/kaf — another Go CLI in the kcat spirit; use when you want a single static binary with a friendlier UX and some admin verbs.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2014-04-07 | First tagged release, as kafkacat[^3]. |
| 1.3.0 | 2016-05-27 | Format strings, JSON output maturing. |
| 1.4.0 | 2019-04-08 | High-level balanced consumer (`-G`) era. |
| 1.5.0 | 2019-09-12 | Timestamp offsets, headers support. |
| 1.6.0 | 2020-07-21 | Transactions, mock cluster (`-M`), broader deserializers. |
| 1.7.0 | 2021-08-23 | Last tagged release; project renamed kafkacat → kcat[^2]. |
| 1.7.1 | 2021 | Docker image tag in current README; the commonly-pulled build. |

## References

[^1]: kcat README — "generic non-JVM producer and consumer for Apache Kafka >=0.8." https://github.com/edenhill/kcat#readme
[^2]: kcat README, "What happened to kafkacat?" — renamed August 2021 for ASF trademark compliance. https://github.com/edenhill/kcat#what-happened-to-kafkacat
[^3]: GitHub API repo + releases metadata for edenhill/kcat (created 2014-03-30, last push 2024-07-09, latest tagged release 1.7.0). https://github.com/edenhill/kcat/releases
[^4]: librdkafka, the underlying Kafka C/C++ client. https://github.com/confluentinc/librdkafka

## Tags

kafka, cli, c, command-line, producer, consumer, librdkafka, streaming, message-queue, avro, netcat, devops
