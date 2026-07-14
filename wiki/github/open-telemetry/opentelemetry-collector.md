# open-telemetry/opentelemetry-collector

> Vendor-agnostic pipeline that receives, processes, and exports traces, metrics, and logs — the reference implementation of the OpenTelemetry data plane.

[GitHub repo](https://github.com/open-telemetry/opentelemetry-collector) ·
[Official website](https://opentelemetry.io) ·
[License: Apache-2.0](https://github.com/open-telemetry/opentelemetry-collector/blob/main/LICENSE)

## Overview

The OpenTelemetry Collector is a standalone service, written in Go, that ingests
telemetry in many formats, runs it through a configurable pipeline, and forwards
it to one or more backends. It exists so that applications emit telemetry once
(ideally as OTLP) and operators decide downstream where it goes — Jaeger,
Prometheus, Loki, or any commercial observability vendor — without recompiling
or re-instrumenting anything. OpenTelemetry itself formed in 2019 from the merger
of OpenTracing and OpenCensus[^1]; the Collector inherited its lineage from the
OpenCensus Service.

The defining tension of this project is **core vs. contrib**. This repository is
the *core*: the pipeline runtime, the internal data model (`pdata`), the OTLP
receiver/exporter, and a small set of processors. Almost every real deployment
also needs components that live in the separate
`open-telemetry/opentelemetry-collector-contrib` repo — vendor exporters,
scrapers (host metrics, Kubernetes, Prometheus), the `filelog` receiver, tail
sampling, and hundreds more[^2]. Understanding which repo a component ships in,
and building a binary that contains exactly the components you need, is the first
real hurdle for new users.

The second defining trait is **versioning**. The Collector distribution has spent
years in `0.x`, shipping a new minor version roughly every four weeks, and those
releases regularly carry breaking config or component changes. Some foundational
modules (notably `pdata` and core config primitives) reached `1.0` stability
independently, but the assembled Collector remains pre-1.0[^3]. The star count on
this repo understates the project's real footprint: it is one of the busiest CNCF
projects, split across core, contrib, and releases repositories.

## Getting Started

Run a prebuilt distribution via Docker, or download a release binary:

```bash
docker run -p 4317:4317 -p 4318:4318 \
  -v $(pwd)/config.yaml:/etc/otelcol/config.yaml \
  otel/opentelemetry-collector:latest
```

A minimal `config.yaml` — receive OTLP, guard memory, batch, export to a backend:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  batch: {}

exporters:
  otlp:
    endpoint: backend.example.com:4317
  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp, debug]
```

The `debug` exporter (which superseded the older `logging` exporter) prints data
to stdout for local verification. Nothing flows until a component is referenced
inside a `service::pipelines` block — declaring a receiver alone does nothing.

## Architecture / How It Works

The pipeline has four component kinds, wired declaratively in YAML:

1. **Receivers** — accept or scrape incoming data (`otlp`, `prometheus`, `filelog`…).
2. **Processors** — transform data in-flight; they run in the exact order listed, and order matters (`memory_limiter` first, `batch` late).
3. **Exporters** — send data out (`otlp`, `otlphttp`, vendor-specific).
4. **Connectors** — added later in the project's life, a connector is an exporter on one pipeline and a receiver on another, letting you chain or fan out pipelines (e.g. `spanmetrics` derives metrics from spans)[^4].

**Extensions** (health check, `pprof`, `zpages`, storage) provide capabilities
that are not part of the data path. A single service can run many independent
pipelines keyed by signal (`traces`, `metrics`, `logs`) and name.

Internally, all data is normalized into **`pdata`**, a Go representation of the
OTLP protobuf model. Every component reads and writes `pdata`, which is what makes
arbitrary receiver/exporter combinations work: the OTLP schema is the lingua
franca. `pdata` uses pooling and careful memory layout because it sits on the hot
path of every signal.

Custom binaries are assembled with the **OpenTelemetry Collector Builder (`ocb`)**,
a code generator that takes a manifest of desired modules and emits a `main.go`
plus a compiled binary containing only those components[^5]. This is the intended
way to run in production — the published "core" and "contrib" images are
convenience bundles, and contrib in particular is large because it links every
community component.

## Production Notes

**Always run `memory_limiter`.** Without it, a downstream backend slowdown causes
the Collector's queues to grow until the process is OOM-killed. `memory_limiter`
applies backpressure by refusing data before that happens. Place it first in the
processor list so it sheds load before expensive work.

**The default sending queue is in-memory and lossy.** On crash or restart, queued
data that has not been exported is lost. Durability requires the `file_storage`
extension (in contrib) wired to each exporter's `sending_queue.storage` — this is
easy to overlook and only matters when it is already too late.

**Core alone is rarely enough.** Scraping host/Kubernetes metrics, reading log
files, tail-based sampling, and every vendor exporter live in contrib. Teams
either run the contrib image (large, links everything) or build a trimmed binary
with `ocb`. Pinning the exact set of components you use reduces attack surface and
image size materially.

**Upgrade churn is real.** With a new minor release roughly monthly, config keys
get renamed, components graduate through alpha/beta/stable, and occasional
breaking changes land. Read the CHANGELOG before bumping, and pin the exact
version in production rather than tracking `latest`. Component stability is tracked
per-component, not for the binary as a whole[^3].

**Tail sampling is stateful and doesn't scale horizontally by default.** The
`tail_sampling` processor (contrib) must see all spans of a trace on one instance,
so a naive load-balanced fleet breaks it. The standard pattern is a two-tier
deployment: a `loadbalancing` exporter routes complete traces by trace ID to a
second tier that does the sampling.

**Cardinality and metrics memory.** High-cardinality attributes (user IDs, URLs
with path params) inflate metric series and Collector memory. Use the
`transform`/`attributes` processors to drop or aggregate labels before they reach
a metrics backend.

**Deployment modes.** The same binary runs as an **agent** (sidecar or node
DaemonSet, close to the workload) or a **gateway** (standalone fleet receiving
from agents). Gateways centralize egress, sampling, and credentials; agents handle
local collection. Most nontrivial setups use both tiers.

## When to Use / When Not

**Use when:**
- You want to decouple instrumentation from backend choice, or send the same telemetry to several backends at once.
- You need to normalize heterogeneous formats (Jaeger, Zipkin, Prometheus, statsd, OTLP) into one pipeline.
- You want processing (batching, filtering, redaction, sampling, enrichment) outside application code.
- You are standardizing on OpenTelemetry across a fleet and need an agent/gateway data plane.

**Avoid when:**
- You have a single app and a single backend that already speaks OTLP — export directly and skip the extra hop.
- You only ship logs and want the lightest possible footprint — a dedicated log shipper is smaller.
- You cannot tolerate the operational cost of tracking a fast-moving `0.x` project and its core/contrib split.

## Alternatives

- grafana/alloy — Grafana's OTel-compatible collector distribution (successor to Grafana Agent); use it when you are already in the Grafana/Prometheus stack and want their component set.
- vectordotdev/vector — Rust-based observability pipeline; use it when logs/metrics throughput and low resource use dominate and you don't need full OTLP trace fidelity.
- fluent/fluent-bit — very lightweight log/metric forwarder in C; use it when edge footprint is the priority and OTLP is secondary.
- fluent/fluentd — mature log aggregation with a huge plugin ecosystem; use it for log-centric pipelines predating OTel.
- prometheus/prometheus — use it directly when your need is metrics scraping and storage, not multi-signal routing.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-05 | Repository created; OpenTracing + OpenCensus merge into OpenTelemetry[^1]. |
| 0.x | 2020– | Monthly minor releases begin; pipeline model (receivers/processors/exporters) stabilizes. |
| 0.x | 2023 | Connectors introduced, enabling pipeline-to-pipeline chaining[^4]. |
| 1.0 (modules) | 2024 | `pdata` and core config modules reach 1.0; assembled Collector stays pre-1.0[^3]. |

## References

[^1]: OpenTelemetry announcement — merger of OpenTracing and OpenCensus. https://opentelemetry.io/community/
[^2]: OpenTelemetry Collector Contrib repository. https://github.com/open-telemetry/opentelemetry-collector-contrib
[^3]: Collector component stability and versioning. https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/component-stability.md
[^4]: Collector configuration — connectors. https://opentelemetry.io/docs/collector/configuration/#connectors
[^5]: OpenTelemetry Collector Builder (ocb). https://opentelemetry.io/docs/collector/custom-collector/

## Tags

go, observability, telemetry, opentelemetry, otlp, tracing, metrics, logging, monitoring, cncf, data-pipeline, agent
