# influxdata/telegraf

> The plugin-everything metrics agent: one static Go binary that collects, transforms, and ships metrics and logs through 300+ compiled-in plugins.

[GitHub repo](https://github.com/influxdata/telegraf) ·
[Official website](https://influxdata.com/telegraf) ·
[License: MIT](https://github.com/influxdata/telegraf/blob/master/LICENSE)

## Overview

Telegraf is InfluxData's collection agent: a single Go binary that gathers metrics, logs, and events from inputs (system stats, SNMP, Modbus, OPC UA, Kafka, Prometheus endpoints, Windows perf counters), optionally transforms them, and writes them to outputs (InfluxDB, Prometheus, Kafka, Graphite, cloud monitoring backends, plain files)[^1]. Started in 2015 as the companion collector for InfluxDB, it outgrew that role — a large share of deployments today never touch InfluxDB and use Telegraf purely as a general-purpose telemetry shovel. At ~17.7k stars and ~5.8k forks with commits landing days before this writing, it is one of the most actively maintained agents in the observability space: minor releases on a roughly quarterly cadence, patches every few weeks[^2].

The defining tradeoff is the **compile-time plugin model**. Every plugin — 300+ of them — is compiled into one dependency-free static binary. That gives trivially simple deployment (copy binary, write TOML, run) and is why Telegraf thrives on IoT gateways, Windows fleets, and air-gapped industrial networks. The cost is a binary well over 100 MB, a supply-chain surface of hundreds of Go dependencies you mostly don't use, and no way to add a plugin without recompiling or shelling out to an external process.

The second tension is stewardship: Telegraf is vendor-neutral in capability (dozens of non-InfluxDB outputs) but single-vendor in governance. InfluxData has kept it MIT-licensed and community-driven (1,200+ contributors[^1]) through the company's own database rewrites, but roadmap priority naturally tracks the InfluxDB ecosystem.

## Getting Started

```bash
# macOS (Debian/RHEL/Windows/Docker: see docs/INSTALL_GUIDE.md)
brew install telegraf
telegraf config > telegraf.conf   # generate a full sample config
```

A minimal pipeline — host metrics to InfluxDB v2, TOML-configured:

```toml
[agent]
  interval = "10s"          # collection cadence
  flush_interval = "10s"    # output write cadence

[[inputs.cpu]]
  percpu = false
  totalcpu = true

[[inputs.mem]]

[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "$INFLUX_TOKEN"
  organization = "my-org"
  bucket = "telegraf"
```

```bash
telegraf --config telegraf.conf --test   # one-shot collect, print, exit
telegraf --config telegraf.conf          # run the agent
```

## Architecture / How It Works

Telegraf's pipeline has exactly four plugin types, executed in order[^3]:

1. **Inputs** — gather metrics. Two flavors: polled inputs run on the agent `interval`; *service inputs* (statsd, `http_listener_v2`, `kafka_consumer`, syslog) run persistent listeners and push metrics in as they arrive.
2. **Processors** — streaming transforms (rename, regex, type conversion, enrichment). The `starlark` processor embeds a Python-like scripting language for custom logic without recompiling. Processors run in config-file order unless an explicit `order` field pins them — an underused knob that bites when transforms depend on each other.
3. **Aggregators** — windowed reductions (min/max/mean, histograms) emitted every `period`. They see metrics *after* processors and re-emit into the stream, which surprises people composing them with processors.
4. **Outputs** — serialize and write. Each output owns an in-memory buffer (`metric_buffer_limit`, default 10,000 metrics) and writes batches (`metric_batch_size`, default 1,000) every `flush_interval`.

The internal metric model is InfluxDB line-protocol-shaped: measurement name, tag set (indexed strings), field set (typed values), timestamp. Parsers (`data_format = "json"`, `"xpath"`, `"prometheus"`, …) map foreign formats into this model on input; serializers map out of it on output. This model is the lingua franca that makes N-inputs-to-M-outputs routing work, but it also means Prometheus labels, OTLP resources, and structured logs all get flattened through a tags/fields view of the world.

Plugins register themselves via Go `init()` into a global registry; the shipped binary imports the `all` packages. Two escape hatches exist for the monolith: the `execd` input/output/processor plugins run any external process speaking line protocol over stdin/stdout (the official external-plugin mechanism)[^4], and a custom-builder tool can compile a trimmed binary containing only the plugins your config references.

## Production Notes

**Backpressure is drop-oldest, per output.** When an output is down longer than `metric_buffer_limit` covers, Telegraf silently discards the oldest buffered metrics. The buffer is RAM-only by default — a restart loses it. Size the limit against your longest tolerable outage, watch the agent's own `internal` input metrics (buffer size, dropped counts), and remember it is per output: five outputs buffer independently, and memory scales accordingly.

**One config, one agent, no isolation.** All plugins share a process. A misbehaving input (an SNMP walk against a slow device, an `exec` script that hangs) delays collection cycles for everything in that config. Operators of large configs commonly split into multiple Telegraf instances or systemd units loading from `telegraf.d/` directories, trading memory for blast-radius isolation.

**Cardinality is your problem.** Telegraf will happily tag metrics with container IDs, ephemeral pod names, or user IDs and forward the explosion downstream. `tagexclude`/`taginclude` modifiers and processors are the tools; nothing warns you by default.

**Config reload is coarse.** `SIGHUP` (or `--watch-config`) reloads the config, but it restarts plugins rather than diffing changes — in-flight service-input state and unflushed buffers are at risk during reload. Treat reloads like restarts.

**Upgrade pains are plugin-level, not agent-level.** The agent core is conservative and 1.x has held compatibility since 2016, but individual plugins get deprecated, renamed, or change field/tag output between minors (the changelog flags these). Pin versions and read release notes for every plugin you actually use — with 300+ plugins most entries won't concern you, but the two that do can break dashboards silently.

**Windows and industrial niches are first-class.** Event Log, WMI, performance counters, Modbus, and OPC UA support are more mature here than in most alternatives — often the deciding factor for Telegraf in OT and enterprise-Windows environments.

## When to Use / When Not

**Use when:**
- You need one agent covering hosts, SNMP devices, industrial protocols (Modbus/OPC UA), message queues, and Windows internals with zero runtime dependencies.
- You ship to InfluxDB — Telegraf is the paved road.
- Config-file-only operation matters: no code, no sidecars, TOML in, metrics out.
- You operate air-gapped or edge fleets where a copy-one-binary deploy story wins.

**Avoid when:**
- Traces matter: Telegraf has no tracing story; the OpenTelemetry Collector is the standard there.
- You are all-in on Prometheus pull-based scraping — adding a push agent adds a second model to reason about.
- Log processing is the primary workload: Telegraf handles logs, but Fluent Bit and Vector have richer parsing/routing and smaller footprints.
- Binary size or dependency-audit surface is a hard constraint and you can't adopt the custom-builder workflow.

## Alternatives

- open-telemetry/opentelemetry-collector — use instead when you want the vendor-neutral OTLP standard and traces + metrics + logs in one pipeline.
- prometheus/node_exporter — use instead when you are Prometheus-native and want minimal pull-based host metrics, not a processing pipeline.
- vectordotdev/vector — use instead for high-throughput log/metric pipelines with programmable transforms (VRL) and lower memory per event.
- fluent/fluent-bit — use instead for log-first collection on constrained edge/Kubernetes nodes (C, single-digit-MB footprint).
- grafana/alloy — use instead when your backend is the Grafana LGTM stack and you want native Prometheus/Loki/OTLP semantics.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2015 | Initial release as InfluxDB's collection agent[^5]. |
| 1.0 | 2016-09 | First stable release; 1.x compatibility line begins. |
| 1.1 | 2016-11 | Processor and aggregator plugin types added. |
| 1.14 | 2020-03 | `execd` external-plugin mechanism[^4]. |
| 1.15 | 2020-07 | Starlark processor for scripted transforms. |
| 1.38.0 | 2026-03-09 | Quarterly minor cadence continues[^2]. |
| 1.39.1 | 2026-06-29 | Latest release at time of writing[^2]. |

## References

[^1]: Telegraf README. https://github.com/influxdata/telegraf/blob/master/README.md
[^2]: Telegraf releases. https://github.com/influxdata/telegraf/releases
[^3]: Telegraf configuration docs. https://github.com/influxdata/telegraf/blob/master/docs/CONFIGURATION.md
[^4]: Telegraf external plugins / `execd`. https://github.com/influxdata/telegraf/blob/master/docs/EXTERNAL_PLUGINS.md
[^5]: Repository created 2015-04-01 (GitHub API metadata).

## Tags

go, metrics, monitoring, observability, time-series, agent, telemetry-collection, influxdb, iot, plugins, logs
