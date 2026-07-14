# fluent/fluentd

> A pluggable log and event collector that decouples data sources from destinations through a unified, tag-routed pipeline.

[GitHub repo](https://github.com/fluent/fluentd) ·
[Official website](https://www.fluentd.org) ·
[License: Apache-2.0](https://github.com/fluent/fluentd/blob/master/LICENSE)

## Overview

Fluentd is a log collector and event router written in Ruby, created by Sadayuki Furuhashi at Treasure Data and first released in 2011[^1]. Its pitch is the "unified logging layer": instead of every application writing to every backend directly, applications emit structured events to Fluentd, which parses, buffers, filters, and forwards them to files, databases, object stores, search engines, and SaaS destinations. The unit of data is a three-part event — tag, timestamp, and a record (a JSON-like map) — serialized internally with MessagePack.

The project is a graduated CNCF project (donated 2016, graduated 2019)[^2] and is one of the two canonical open-source log shippers alongside Elastic's Logstash. It is the "F" in the EFK stack (Elasticsearch + Fluentd + Kibana). Its defining strength is a plugin ecosystem of 1,000+ community plugins covering nearly every input and output you might need; its defining tension is that it is a Ruby process, so raw single-core throughput and memory footprint are worse than a compiled collector — which is exactly why the same org also maintains Fluent Bit (C), a lighter sibling now more common as the node-level agent.

Fluentd is mature and slow-moving in the good sense: the v1 configuration format and plugin API have been stable since late 2017, and most operational knowledge from years ago still applies. New feature velocity is low; correctness and plugin breadth are the point.

## Getting Started

```bash
gem install fluentd
fluentd --setup ./fluent      # writes a sample fluent.conf
fluentd -c ./fluent/fluent.conf &
echo '{"json":"message"}' | fluent-cat debug.test
```

A minimal config that tails a log file, tags events, and routes them to stdout and to files:

```apache
# fluent.conf
<source>
  @type tail
  path /var/log/app/*.log
  pos_file /var/log/fluentd/app.pos
  tag app.access
  <parse>
    @type json
  </parse>
</source>

<filter app.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>

<match app.**>
  @type file
  path /var/log/fluentd/archive/app
  <buffer>
    @type file
    flush_interval 10s
    retry_max_times 5
  </buffer>
</match>
```

Tags flow left to right: `<source>` assigns a tag, `<filter>` mutates matching events, `<match>` sends them to an output. The `**` glob and first-match-wins ordering are the whole routing model.

## Architecture / How It Works

Fluentd is a single Ruby process (optionally multi-worker) built around an event-driven core and a plugin taxonomy. Every moving part is a plugin:

- **Input** (`@type tail`, `forward`, `http`, `syslog`, `tail`, `exec`) — produce tagged events.
- **Parser** — turn raw bytes into structured records (json, regexp, ltsv, apache2, csv, multiline).
- **Filter** (`record_transformer`, `grep`, `parser`) — mutate or drop events in the pipeline.
- **Output** (`file`, `forward`, `s3`, `elasticsearch`, `kafka`, `stdout`) — write to destinations.
- **Buffer** and **Formatter** — pluggable chunking/serialization attached to outputs.

Events are matched to outputs by tag using glob patterns; the first `<match>` whose pattern matches wins, so ordering in the config file is semantically significant. Internally events carry a MessagePack-encoded record, which is why binary-safe, schema-loose payloads pass through cheaply.

The core reliability mechanism is the **buffer**. Output plugins that inherit from the buffered output base accumulate events into chunks (keyed optionally by tag or a time/field slice), then flush chunks to the destination. Buffers are `memory` (fast, lost on crash) or `file` (survives restart, bounded by disk). Failed flushes retry with exponential backoff up to `retry_max_times`, after which the chunk goes to a secondary output or is dropped. `flush_interval`, `chunk_limit_size`, `total_limit_size`, and the retry parameters are the knobs that decide your latency/durability/backpressure tradeoff.

The v1 API (2017) introduced **multi-process workers** (`<system> workers N`), a redesigned plugin base-class hierarchy, and a clearer buffer model, superseding the v0.12 API. Because Ruby has a global VM lock, a single worker is effectively single-core for Ruby-level work; horizontal scaling within one host is done by running multiple workers and sharding inputs across them.

## Production Notes

**Throughput and the GIL.** One worker saturates one core. High-volume nodes need `workers N` plus inputs that can be partitioned (e.g. multiple `tail` sources, or `forward` fan-in). If you are CPU-bound on a busy Kubernetes node, the common answer is to move node-level collection to Fluent Bit and keep Fluentd as an aggregation tier.

**Buffer sizing is the #1 operational footgun.** With file buffers, an unreachable destination fills disk; with memory buffers, a backlog is lost on restart and can OOM the process. Always set `total_limit_size` and decide the `overflow_action` (block vs drop). `retry_forever` plus a down backend is a classic way to silently accumulate chunks forever.

**`in_tail` state.** File tailing tracks position in a `pos_file`. Log rotation, truncation, symlink swaps, and very short-lived files all have edge cases; `read_from_head`, `follow_inodes`, and `pos_file_compaction_interval` exist specifically to paper over rotation bugs. Missing or shared `pos_file` between workers causes duplicate or lost reads.

**Plugin quality is uneven.** The 1,000+ plugins are community-maintained; some lag Fluentd core releases or the destination's API. The Elasticsearch/OpenSearch output plugins in particular have a history of breaking on version bumps of the target cluster. Pin plugin versions and test upgrades.

**Packaging and versions.** The vendor distribution was `td-agent` (Treasure Data's packaged Fluentd + curated plugins + bundled Ruby). It was rebranded to `fluent-package` (v5, 2023); `td-agent` v4 is end of life[^3]. New deployments should use `fluent-package` or the official container images rather than plain `gem install` if they want a supported, batteries-included build. v0.12 has been unsupported since 2019[^4].

**Ruby version.** Current master requires Ruby 3.2 or later. Running Fluentd on the system Ruby of an old distro is a common source of gem-native-extension build failures; use the packaged distribution's bundled Ruby.

## When to Use / When Not

**Use when:**
- You need broad destination coverage (S3, Kafka, Elasticsearch, BigQuery, Splunk, etc.) with buffering and retry out of the box.
- You want an aggregation tier that many lightweight forwarders fan into over the `forward` protocol.
- You value routing/transform flexibility and a stable, well-documented config format over raw throughput.
- You're building an EFK logging stack and want the reference collector.

**Avoid when:**
- You need the smallest possible per-node footprint on thousands of containers — reach for Fluent Bit.
- You're standardizing on OpenTelemetry end-to-end and want traces/metrics/logs in one collector — the OTel Collector is a better fit.
- Your workload is a single high-throughput pipeline where a compiled collector's per-core efficiency matters (Vector, Fluent Bit).
- You want push-down storage and query, not just shipping — Fluentd routes, it does not store or index.

## Alternatives

- fluent/fluent-bit — same org, C, tiny footprint; use instead as the node/edge agent, often forwarding to a Fluentd aggregator.
- vectordotdev/vector — Rust collector with strong per-core throughput and a typed transform language; use when performance and a unified config matter more than plugin breadth.
- elastic/logstash — JVM-based, richer built-in transforms; use if you're already all-in on the Elastic stack and accept the heavier runtime.
- open-telemetry/opentelemetry-collector — vendor-neutral logs/metrics/traces; use when you want one collector across all telemetry signals.
- grafana/loki — not a collector but a log store/query engine; use alongside a shipper when the gap you're filling is storage, not routing.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.10 | 2011 | Initial public releases; unified logging layer concept[^1]. |
| 0.12 | 2015 | Long-lived stable line; filter directive, label routing. Now EOL[^4]. |
| 1.0 | 2017-12 | New plugin API, multi-process workers, redesigned buffering[^5]. |
| — | 2019-04 | Graduated within CNCF[^2]. |
| fluent-package 5.0 | 2023 | td-agent rebrand; bundled Ruby 3.x, curated plugins[^3]. |

## References

[^1]: Sadayuki Furuhashi, "Fluentd: the missing log collector." Treasure Data / Fluentd project origin. https://www.fluentd.org/architecture
[^2]: CNCF, "Fluentd" graduated project page. https://www.cncf.io/projects/fluentd/
[^3]: Fluentd blog, "Migrating from td-agent to fluent-package." https://www.fluentd.org/blog/td-agent-will-be-renamed
[^4]: Fluentd blog, "Drop schedule announcement in 2019" (v0.12 end of support). https://www.fluentd.org/blog/drop-schedule-announcement-in-2019
[^5]: Fluentd v1.0 documentation and plugin API v1. https://docs.fluentd.org/

## Tags

ruby, logging, log-collector, observability, cncf, data-pipeline, log-aggregation, plugins, buffering, efk-stack, infrastructure
