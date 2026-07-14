# jaegertracing/jaeger

> Distributed tracing backend — collect, store, and query spans from instrumented services, with a web UI for trace search and dependency analysis.

[GitHub repo](https://github.com/jaegertracing/jaeger) ·
[Official website](https://www.jaegertracing.io/) ·
[License: Apache-2.0](https://github.com/jaegertracing/jaeger/blob/main/LICENSE)

## Overview

Jaeger is a distributed tracing platform originally built at Uber (2015–2016) and donated to the Cloud Native Computing Foundation, which it entered as an incubating project in 2017 and graduated in October 2019 as the 7th CNCF top-level project[^1]. It solves one problem: reconstructing the causal path of a single request as it fans out across many services, so you can see where latency and errors actually originate. It is a backend — the collection, storage, and query tiers plus a UI — not an instrumentation library.

The defining shift in Jaeger's history is its relationship to OpenTelemetry. Jaeger's own client SDKs and the OpenTracing standard it was built on are deprecated; instrumentation is expected to come from OpenTelemetry SDKs speaking OTLP[^2]. Jaeger v2, released in late 2024, takes this to its conclusion: the entire backend is rebuilt as a distribution of the OpenTelemetry Collector, with Jaeger's collector, query, and storage logic packaged as Collector components rather than standalone binaries[^3]. This makes Jaeger a specialized configuration of a general pipeline rather than a self-contained system.

The central tradeoff: Jaeger is a mature, indexed trace store with strong search, but it is head-sampling-oriented and its operational weight lives entirely in the storage backend you choose. It stores traces well; it does not decide for you what to store, and it does not run the database.

## Getting Started

The all-in-one image bundles collector, query, UI, and in-memory storage for local use:

```bash
docker run --rm --name jaeger \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/jaeger:latest
# UI at http://localhost:16686
# Send OTLP: gRPC on 4317, HTTP on 4318
```

Point an OpenTelemetry SDK at it (Python shown):

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="localhost:4317", insecure=True))
)
trace.set_tracer_provider(provider)

with trace.get_tracer(__name__).start_as_current_span("checkout"):
    ...  # traced work; appears in the Jaeger UI under this service
```

In-memory storage is not durable — it exists to demo the UI. Any real deployment needs an external backend (below).

## Architecture / How It Works

A Jaeger deployment is a pipeline of separable roles:

- **Collector** — receives spans (OTLP over gRPC/HTTP is the current path), validates and transforms them, applies sampling policy, and writes to storage. Stateless and horizontally scalable.
- **Storage** — the pluggable backend. Officially supported: **Cassandra**, **Elasticsearch / OpenSearch**, in-memory (all-in-one only), and **Badger** (embedded key-value store for single-node local disk). Kafka can sit between collector and storage as a buffer.
- **Ingester** — when Kafka is used, the ingester consumes from the topic and writes to the durable store, decoupling ingest spikes from storage write throughput.
- **Query** — reads from storage and serves the API the UI consumes.
- **UI** — a separate React app (`jaegertracing/jaeger-ui`) for trace search, timeline (Gantt) views, and service dependency graphs.

**Sampling is head-based by default.** The decision to keep or drop a trace is made at the start, at the SDK, using probabilistic or rate-limiting strategies — optionally distributed from the collector via *remote sampling*[^4]. The consequence is structural: you cannot decide to keep a trace *because* it errored or was slow, since that is only known after the fact. Tail-based sampling exists only via the OpenTelemetry Collector's `tail_sampling` processor, which under Jaeger v2 you configure as part of the same Collector pipeline.

**v1 vs v2.** Jaeger v1 was a set of purpose-built Go binaries (agent, collector, query, ingester). The **agent is removed** in v2, and configuration moves from Jaeger-specific CLI flags to OpenTelemetry Collector YAML[^3]. Functionally the two are similar; operationally the config surface and component model are different, and this is the migration everyone on v1 eventually faces.

## Production Notes

**The storage backend is the deployment.** Jaeger's own components are stateless and easy; the hard operational work is running Cassandra or Elasticsearch at trace volume. Both need real capacity planning — traces are high-volume, write-heavy, short-retention data. On Elasticsearch/OpenSearch you manage index rotation and lifecycle (ILM/ISM) and the `es-index-cleaner`/rollover jobs, or indices grow unbounded. On Cassandra you manage compaction and TTL. Neither is set-and-forget.

**Head sampling caps observability.** With probabilistic sampling at, say, 1%, the error trace you want is probably not stored. Teams routinely discover this during an incident. The fixes — raising sample rates (cost), or moving to tail sampling via the Collector (added memory/latency, buffering all spans of a trace until the decision) — both have real cost. Decide the sampling strategy before you need the data, not after.

**Instrumentation must be OpenTelemetry.** Jaeger's client libraries are deprecated and should not be used in new code[^2]. If you have legacy `jaeger-client-*` or OpenTracing instrumentation, plan a migration to OTel SDKs; Jaeger still ingests older formats but this path is on borrowed time.

**Service Performance Monitoring (the Monitor / SPM tab)** is not self-contained: it needs span metrics (RED-style) produced by the OpenTelemetry `spanmetrics` connector and stored in a Prometheus-compatible backend that Query is configured to read. Expect to stand up Prometheus separately.

**Kafka for burst protection.** Direct collector→storage coupling means storage backpressure becomes ingest loss under load. The Kafka + ingester topology decouples them and is the standard answer for high-throughput or spiky workloads, at the cost of running Kafka.

**Retention is short by design.** Traces are typically kept days, not months. Long-term trace retention is expensive on indexed stores — this is precisely the gap object-storage backends like Tempo target.

## When to Use / When Not

**Use when:**
- You want an open-source, self-hosted trace backend with strong search and a mature UI.
- Your instrumentation is (or is becoming) OpenTelemetry/OTLP.
- You already operate Cassandra or Elasticsearch and can give traces a home there.
- You want CNCF-graduated governance and no per-span vendor billing.

**Avoid when:**
- You don't want to run and tune a storage database — a managed tracing vendor or an object-storage backend is less operational load.
- You need cheap, long-retention trace storage at very high volume (object-storage designs fit better).
- You need tail-based sampling as a first-class, turnkey feature rather than a Collector processor you assemble.
- You want metrics, logs, and traces in one unified store — Jaeger is traces only.

## Alternatives

- grafana/tempo — use instead when you want cheap, high-volume trace storage on object storage (S3/GCS) with TraceQL and no index database to operate; weaker ad-hoc search than Jaeger.
- openzipkin/zipkin — use instead when you want the older, simpler, JVM-centric tracing backend with a smaller moving-parts footprint.
- open-telemetry/opentelemetry-collector — not a competitor but the substrate Jaeger v2 is built on; use it directly when you need a vendor-neutral pipeline that fans out to multiple backends.
- SigNoz/signoz — use instead when you want traces, metrics, and logs in one ClickHouse-backed product rather than a traces-only backend.
- elastic/apm-server — use instead when you are already committed to the Elastic Stack and want tracing integrated with its logs and APM.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-04 | Repository opened; originated from Uber's internal tracing work[^1]. |
| CNCF incubation | 2017-09 | Accepted as a CNCF incubating project[^1]. |
| 1.0 | 2019 | v1 line: agent/collector/query/ingester binaries. |
| Graduated | 2019-10 | 7th CNCF top-level graduated project[^1]. |
| OTLP support | ~2022 | Native OTLP ingestion; Jaeger client SDKs deprecated[^2]. |
| 2.0 | 2024-11 | Rebuilt as an OpenTelemetry Collector distribution; agent removed[^3]. |

## References

[^1]: CNCF, "Cloud Native Computing Foundation Announces Jaeger Graduation" — 2019-10-31. https://www.cncf.io/announcement/2019/10/31/cloud-native-computing-foundation-announces-jaeger-graduation/
[^2]: Jaeger project, "Jaeger clients — deprecation and migration to OpenTelemetry." https://www.jaegertracing.io/docs/latest/client-libraries/
[^3]: Jaeger project, "Jaeger v2 released" — 2024. https://medium.com/jaegertracing/jaeger-v2-released-09a6033d1b10
[^4]: Jaeger project, "Sampling" documentation. https://www.jaegertracing.io/docs/latest/sampling/

## Tags

go, distributed-tracing, observability, opentelemetry, cncf, tracing, otlp, backend, cassandra, elasticsearch
