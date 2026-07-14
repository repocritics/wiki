# vectordotdev/vector

> A Rust observability pipeline that collects, transforms, and routes logs and metrics between arbitrary sources and vendors.

[GitHub repo](https://github.com/vectordotdev/vector) ·
[Official website](https://vector.dev) ·
[License: MPL-2.0](https://github.com/vectordotdev/vector/blob/master/LICENSE)

## Overview

Vector is an end-to-end observability data pipeline written in Rust. It sits between the systems that produce telemetry (files, journald, Kafka, syslog, container runtimes, cloud APIs) and the systems that store or analyze it (Elasticsearch, Datadog, S3, Loki, ClickHouse, any HTTP endpoint), and lets operators reshape, filter, sample, and re-route that data in flight. It was created by Timber.io (Ben Johnson, Luke Steensen) and open-sourced in 2019; Datadog acquired Timber.io in February 2021 and now maintains Vector through its Community Open Source Engineering team[^1].

The pitch is control and cost: because you own the pipeline, you can drop noisy fields, aggregate logs into metrics, or fan the same stream out to a cheap archive and an expensive index simultaneously — deciding where data lands rather than accepting a vendor's defaults. The recurring counter-pressure is the OpenTelemetry Collector, which has become the CNCF-blessed vendor-neutral default; Vector's differentiation is raw throughput, a mature sink catalog, and VRL, at the cost of not being OTel-native.

Note the license: **MPL-2.0**, weak copyleft at the file level. This is more permissive than GPL but stricter than the MIT/Apache-2.0 that most Rust infrastructure uses — modifications to Vector's own source files must be shared, though linking and configuration are unaffected. Also note that after years of development Vector still ships under **0.x versioning**; there is no 1.0 stability guarantee, and minor releases carry breaking changes documented in per-release upgrade guides.

## Getting Started

```bash
curl --proto '=https' --tlsv1.2 -sSfL https://sh.vector.dev | bash
```

A minimal pipeline — tail JSON logs, parse them with VRL, print to stdout:

```yaml
# vector.yaml
sources:
  app_logs:
    type: file
    include:
      - /var/log/app/*.log

transforms:
  parse:
    type: remap
    inputs: ["app_logs"]
    source: |
      . = parse_json!(string!(.message))
      .env = "prod"

sinks:
  out:
    type: console
    inputs: ["parse"]
    encoding:
      codec: json
```

```bash
vector validate vector.yaml   # type-check config + VRL before running
vector --config vector.yaml
```

## Architecture / How It Works

A Vector pipeline is a directed graph of three component kinds wired by `inputs` references:

1. **Sources** ingest data (`file`, `journald`, `kafka`, `socket`, `http_server`, `kubernetes_logs`, cloud pull APIs).
2. **Transforms** reshape it (`remap`/VRL, `filter`, `route`, `aggregate`, `sample`, `dedupe`, `reduce`).
3. **Sinks** send it onward (`elasticsearch`, `datadog_logs`, `aws_s3`, `clickhouse`, `loki`, `http`, and ~40 others).

The internal data model is the **event**, of which there are logs, metrics, and (still limited) traces. A recurring friction point is that many upstream tools model metrics as structured logs; Vector has a real metric type, but interoperability across the log/metric boundary is where configuration gets fiddly.

**VRL (Vector Remap Language)** is the centerpiece transform. It is a purpose-built, expression-oriented DSL that compiles to a sandboxed execution engine rather than embedding a general scripting runtime. It exists because the earlier `lua` transform was slow and unsafe under load; VRL is fallibility-aware (functions like `parse_json!` abort on error with `!`, or return errors you handle explicitly) and is checked at `vector validate` time, so many mistakes surface before deploy rather than mid-stream[^2].

The runtime is Rust on the Tokio async executor, with a per-component concurrency model and backpressure that propagates upstream when a sink stalls. **Adaptive Request Concurrency (ARC)** replaces hand-tuned rate limits on HTTP sinks with an AIMD control loop that observes latency and error rates and adjusts in-flight request counts automatically[^3]. **Disk buffers** provide at-least-once delivery across restarts by persisting un-acked events to local disk — but this is node-local durability, not a replicated queue like Kafka.

Vector deploys in two roles from the same binary: **agent** (one per node, tails local logs, minimal transforms) and **aggregator** (a central tier that receives from agents, does heavy transformation, and fans out to sinks)[^4].

## Production Notes

**0.x breaking changes are real.** Because there is no 1.0 contract, minor upgrades can change component defaults, VRL semantics, or config schema. Read the upgrade guide for every version bump; pin exact versions in production and stage upgrades rather than tracking latest.

**Memory and buffers.** Default in-memory buffers bound throughput by RAM and lose in-flight data on crash. Disk buffers trade latency and disk I/O for durability but only protect against process restart, not disk loss — pair with sink retries and end-to-end acknowledgements for the strongest guarantee. High-cardinality metric aggregation (`aggregate`, `tag_cardinality_limit`) is a common memory-blowup source; cap cardinality explicitly.

**VRL has a learning curve.** The fallibility model (`!` vs error-coalescing) trips up newcomers, and complex per-event VRL is CPU-bound — it runs on every event. Profile before pushing heavy parsing into `remap`; prefer structured sources where possible.

**Config reloads.** `SIGHUP` triggers a hot reload, but not every change is reload-safe; some source/sink changes require a restart, and a bad reload can leave a component down. Always `vector validate` in CI and roll out via a supervisor that can revert.

**Kubernetes.** The `kubernetes_logs` source is the standard DaemonSet agent pattern; watch its file-position checkpointing and the memory footprint under log-rotation churn on busy nodes.

**Observability of the pipeline itself.** Vector exposes internal metrics and a GraphQL API (`vector top`, `vector tap`) for live inspection — use `vector tap` to watch events flow through a specific component when a transform silently drops data.

## When to Use / When Not

**Use when:**
- You need to cut observability spend by filtering, sampling, or routing before data hits a metered vendor.
- You are migrating between backends and want to dual-write without touching applications.
- You need high single-node throughput and a broad catalog of ready sinks.
- You want config-driven pipelines with compile-time-validated transformation logic (VRL).

**Avoid when:**
- You want a CNCF-standard, OTel-native collector — the OpenTelemetry Collector is the more future-proof default for greenfield OTel stacks.
- You need replicated, durable queuing — Vector's disk buffers are node-local; put Kafka in front for real durability.
- Traces are central to your use case — trace support remains the least mature of the three data types.
- You need a frozen, 1.0-stable dependency with rare breaking changes.

## Alternatives

- open-telemetry/opentelemetry-collector — use instead when you want the vendor-neutral CNCF standard and OTel-native pipelines over raw throughput.
- fluent/fluent-bit — use instead when you need a tiny C-based agent footprint on constrained nodes.
- fluent/fluentd — use instead when you want the largest mature plugin ecosystem and don't need Rust-level performance.
- elastic/beats — use instead when you are fully inside the Elastic stack and want first-party shippers.
- influxdata/telegraf — use instead when your workload is metrics-first rather than log-first.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019 | Open-sourced by Timber.io, written in Rust[^1]. |
| — | 2021-02 | Datadog acquires Timber.io; Vector maintained by Datadog OSS team[^1]. |
| 0.13–0.14 | 2021 | VRL (Vector Remap Language) becomes the primary transform[^2]. |
| — | 2021 | Adaptive Request Concurrency (ARC) for HTTP sinks[^3]. |
| 0.x | ongoing | Monthly-cadence releases; still pre-1.0, per-release upgrade guides. |

## References

[^1]: Vector README and About page, Datadog Community Open Source Engineering. https://vector.dev/ · https://github.com/vectordotdev/vector
[^2]: VRL (Vector Remap Language) reference. https://vector.dev/docs/reference/vrl/
[^3]: "Adaptive Request Concurrency" architecture docs. https://vector.dev/docs/architecture/arc/
[^4]: Deployment roles — agent vs aggregator. https://vector.dev/docs/setup/deployment/roles/

## Tags

rust, observability, logging, metrics, telemetry, data-pipeline, etl, stream-processing, vrl, datadog, cloud-native, vendor-neutral
