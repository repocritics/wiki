# fluent/fluent-bit

> A C-based telemetry agent for collecting, processing, and forwarding logs, metrics, and traces — the lightweight edge complement to Fluentd.

[GitHub repo](https://github.com/fluent/fluent-bit) ·
[Official website](https://fluentbit.io) ·
[License: Apache-2.0](https://github.com/fluent/fluent-bit/blob/master/LICENSE)

## Overview

Fluent Bit is a telemetry collector written in C, started at Treasure Data around 2015 as a smaller sibling to the Ruby-based Fluentd, originally aimed at embedded and IoT log forwarding[^1]. It is now part of the Fluentd project under the CNCF and is one of the default log-shipping agents in the Kubernetes ecosystem, typically deployed as a per-node DaemonSet[^2]. As of this writing it has ~7,900 stars and ~1,900 forks, with commits landing daily; the active `master` branch targets a v5.1 development line and the project ships major releases roughly every 3–4 months[^3].

The defining tradeoff is footprint versus completeness. Fluent Bit is deliberately lean — no Ruby runtime, low memory, fast startup — which is why it wins at the collection edge (nodes, sidecars, appliances). The cost is that heavy aggregation, the largest plugin catalog, and the most flexible routing historically belonged to Fluentd; the two were designed to be used together (Bit forwards, Fluentd aggregates), though Fluent Bit has grown enough that many deployments now use it end to end. Since v2.0 it is no longer a logs-only tool: it ingests and exports metrics and traces (OpenTelemetry, Prometheus) and markets itself as a general telemetry agent[^4].

## Getting Started

```bash
# Debian/Ubuntu via the official install script
curl https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh | sh
# or Docker
docker run --rm fluent/fluent-bit -i cpu -o stdout -f 1
```

A minimal pipeline in the classic config format — tail a file, tag it, ship matching tags to stdout:

```ini
[SERVICE]
    flush        1
    log_level    info

[INPUT]
    name         tail
    path         /var/log/app/*.log
    tag          app.*
    db           /var/lib/fluent-bit/tail.db   # persist read offsets across restarts

[FILTER]
    name         modify
    match        app.*
    add          host ${HOSTNAME}

[OUTPUT]
    name         stdout
    match        app.*
```

```bash
fluent-bit -c fluent-bit.conf
```

The newer YAML config format is functionally equivalent and is where processor-based pipelines are best expressed[^5].

## Architecture / How It Works

Fluent Bit is a single-process event-loop engine. Data moves through a fixed pipeline: **input → parser → filter → buffer → router → output**. Every record is normalized into MessagePack internally, and every chunk of records carries a **tag**; routing is entirely tag-based — each output declares a `Match` pattern (globs allowed) and receives every chunk whose tag matches. There is no central topology graph, only tag/match wiring, which is powerful but makes large configs hard to reason about.

Concurrency is cooperative. The core runtime uses an event loop with coroutines for non-blocking network and file I/O, so a single thread handles many connections; some plugins and the newer processor path can use worker threads. Inputs, filters, and outputs are plugins compiled in (70+ built-in), and the plugin surface is polyglot: core plugins in C, in-flight transforms in **Lua** filters, and outputs authorable in **Go**[^6]. There is also a WASM path and an SQL stream-processor that runs continuous queries over the record stream.

The buffering layer is the part operators most need to understand. Chunks live in memory by default; setting `storage.type filesystem` backs them with the on-disk chunk-I/O store so data survives restarts and backpressure. Each input can be bounded with `Mem_Buf_Limit`, and when a destination is slow the engine applies **backpressure** by pausing the offending input rather than growing memory without bound. The `tail` input additionally keeps a SQLite `db` of file offsets so it resumes cleanly rather than re-reading or dropping on restart.

## Production Notes

- **Buffering is the #1 source of data loss and OOM.** With the default memory buffering, a slow or unreachable output plus unbounded `Mem_Buf_Limit` will grow the process until the kernel kills it; a bounded limit will instead pause the input and drop new data. Filesystem buffering (`storage.type filesystem` plus `storage.total_limit_size` on outputs) is the durable configuration, and it is not the default — you must opt in.
- **Multiline logs are a recurring footgun.** Container stdout is split into lines by the runtime, so stack traces arrive as separate records. The `multiline` filter / built-in parsers (e.g. `docker`, `cri`, custom rules) address this, but misconfiguration silently ships fragmented logs. Budget real time for this in any Kubernetes rollout.
- **`tail` offset DB requires a writable, persistent path.** In containers the `db` file must live on a mounted volume, or offsets reset on every restart and you re-ingest or lose data.
- **Two config formats coexist.** The classic `.conf` format and the newer YAML format are not 1:1 in ergonomics; processors and some newer features are documented primarily against YAML. Mixing mental models across a fleet causes drift.
- **Lua and SQL transforms are convenient but not free** — per-record Lua callbacks add CPU cost at high throughput; measure before relying on them in hot paths.
- **Version cadence is fast.** With majors every 3–4 months, plugin defaults and behaviors shift more often than an infrastructure agent typically does; pin exact image tags and read release notes before bumping, rather than tracking `latest`.
- **Windows and BSD are supported but less battle-tested** than Linux; most production hardening and plugin coverage assumes Linux.

## When to Use / When Not

**Use when:**
- You need a low-footprint agent on every node/pod/appliance to collect and forward logs (and now metrics/traces).
- You are in Kubernetes and want a CNCF-aligned DaemonSet collector.
- You want vendor-neutral routing to many backends (Elasticsearch/OpenSearch, Loki, S3, Kafka, OTLP, cloud logging services) from one binary.
- Memory and CPU budgets are tight (edge, embedded, dense nodes).

**Avoid when:**
- You need very complex aggregation, buffering strategies, or the largest possible plugin catalog — Fluentd (or a dedicated pipeline) may fit better as the aggregation tier.
- Your telemetry is metrics/traces-first and logs-secondary — the OpenTelemetry Collector is more idiomatic there.
- You want a modern transform DSL and single-binary performance for high-volume observability pipelines — Vector is a direct competitor worth evaluating.

## Alternatives

- fluent/fluentd — the Ruby sibling and historical aggregator tier; use it when you need the largest plugin ecosystem and richer buffering and Fluent Bit as the edge forwarder.
- vectordotdev/vector — Rust agent with the VRL transform language and strong throughput; use it when you want one high-performance binary with a first-class transform DSL.
- open-telemetry/opentelemetry-collector — use it when metrics and traces are primary and you want a vendor-neutral OTel-native pipeline.
- elastic/beats — use Filebeat/Metricbeat when you are committed to the Elastic Stack and want tight integration over neutrality.
- grafana/alloy — use it when you are standardized on Grafana/Loki/Mimir and want a collector aligned to that stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015 | Initial releases; embedded/IoT log forwarder, Fluentd's lightweight edge companion[^1]. |
| 1.0 | 2019 | First stable line; Kubernetes filter, `tail` state DB, broad output plugins[^2]. |
| 1.3–1.9 | 2019–2022 | Filesystem/chunk buffering, storage limits, expanded plugin catalog. |
| 2.0 | 2022-10 | Metrics and traces support, processors; repositioned as a telemetry agent[^4]. |
| 3.0 | 2024 | Continued pipeline/processor and protocol work; YAML config maturity. |
| 4.x | 2025 | Recent major line. |
| 5.1 | 2026 (dev) | Current active `master` development target[^3]. |

## References

[^1]: Fluent Bit — official site and project background. https://fluentbit.io
[^2]: Fluent Bit manual — Kubernetes deployment and filters. https://docs.fluentbit.io/manual/installation/kubernetes
[^3]: Repository metadata and README roadmap (stars/forks, release cadence, v5.1 development), fetched 2026-07-15. https://github.com/fluent/fluent-bit
[^4]: Fluent Bit v2.0 announcement — metrics, traces, and processors. https://fluentbit.io/blog/2022/10/25/fluent-bit-v2-0-0-is-here/
[^5]: Fluent Bit manual — YAML configuration. https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/yaml
[^6]: Fluent Bit manual — plugin development (C, Lua filters, Go outputs). https://docs.fluentbit.io/manual/development

## Tags

c, logging, observability, telemetry, log-forwarder, metrics, tracing, cloud-native, cncf, kubernetes, opentelemetry, stream-processing
