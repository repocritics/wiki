# openzipkin/zipkin

> Distributed tracing in the Google Dapper lineage — a self-contained server that collects, stores, and visualizes timing data to troubleshoot latency in service architectures.

[GitHub repo](https://github.com/openzipkin/zipkin) ·
[Official website](https://zipkin.io) ·
[License: Apache-2.0](https://github.com/openzipkin/zipkin/blob/master/LICENSE)

## Overview

Zipkin is a distributed tracing system open-sourced by Twitter in 2012, explicitly modeled on Google's Dapper paper[^1][^2]. Instrumented applications report timed spans; Zipkin correlates them into traces, answers queries like "where did this request spend its time", and renders a service dependency graph. It predates the modern observability wave by nearly a decade — its B3 propagation headers (`X-B3-TraceId`, `X-B3-SpanId`, ...) were the de facto cross-vendor trace-context standard before W3C `traceparent` existed[^3][^7].

Zipkin is really three things: a wire format (v2 JSON spans), a propagation convention (B3), and a server. The server is deliberately minimal — a single executable jar with an embedded UI, in-memory storage by default, and pluggable persistent backends (Cassandra, Elasticsearch/OpenSearch). This minimalism is the defining tradeoff: Zipkin is the easiest tracing backend to stand up, and one of the least featureful to grow into.

The project's strategic position has shifted. OpenTelemetry absorbed the instrumentation and collection side of the market; most new deployments emit OTLP and treat Zipkin — if at all — as a receiving format or a lightweight backend[^8]. At 17.4k stars, Zipkin remains one of the most widely deployed tracing systems ever, but commit activity has slowed to maintenance cadence (last push April 2026), and the interesting development in this space now happens elsewhere.

## Getting Started

The server requires JRE 17+ and listens on port 9411:

```bash
docker run -d -p 9411:9411 openzipkin/zipkin
# or without Docker:
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

Report a span directly against the v2 API (normally a tracer library like Brave does this):

```bash
curl -X POST http://localhost:9411/api/v2/spans \
  -H 'Content-Type: application/json' \
  -d '[{
    "traceId": "d3d200866a77cc59",
    "id": "d3d200866a77cc59",
    "name": "get /users",
    "timestamp": 1721000000000000,
    "duration": 150000,
    "localEndpoint": { "serviceName": "user-service" }
  }]'
```

Traces appear in the UI at `http://localhost:9411/zipkin`. A `zipkin-slim` image starts faster but drops messaging transports (Kafka, RabbitMQ) and supports only in-memory and Elasticsearch storage.

## Architecture / How It Works

The pipeline is: **instrumentation → transport → collector → storage → query API → UI**. Applications are instrumented with a tracer (Brave for Java is the reference; dozens of community libraries exist for other languages) and report spans over HTTP POST, Kafka, RabbitMQ, ActiveMQ, gRPC, or Pulsar. Collectors decode and write spans through a `StorageComponent` SPI[^6]:

- **In-memory** — default; not persistent, not viable for realistic workloads.
- **Cassandra** — the scale backend (second-generation schema, UDT-based, spans readable as v2 JSON in cqlsh). Tested against Cassandra 4.1.
- **Elasticsearch** — stores spans as v2 JSON documents; tested against ES 7–8.x and OpenSearch 2.x.
- **MySQL (v1, legacy)** — kept only for migration; columns-per-field schema with known, unfixed performance problems[^4].

Two design decisions are worth understanding. First, the **core library** is a dependency-free ~155 KB jar targeting Java 8, so agent authors can embed the model and codecs without conflicts; the server itself requires Java 17 and is built on Armeria as its HTTP engine. Second, the **service dependency graph is not computed online**: for Cassandra and Elasticsearch it requires running a separate Spark job (`zipkin-dependencies`) on a schedule[^5] — a frequent surprise for new operators whose dependency page stays empty.

Search (`GET /traces`, `/services`, `/spans`, autocomplete endpoints) can be disabled with `SEARCH_ENABLED=false`, trading the trace-list UI for lower storage cost and higher write throughput — viable only when trace IDs are discoverable through logs.

## Production Notes

**The default is a demo.** In-memory storage loses everything on restart and degrades under load. Production means Cassandra or Elasticsearch, and each brings its own operational bill.

**Elasticsearch index lifecycle is your problem.** Zipkin writes daily indices but ships no retention management; without ILM policies or curator jobs, disk fills until the cluster degrades. Cassandra deployments similarly own their own TTL and compaction tuning.

**The dependency graph needs a cron'd Spark job.** `zipkin-dependencies` must run periodically (typically daily) against Cassandra/ES; sizing Spark for large span volumes is real work, and the graph is only as fresh as the last run[^5].

**No auth, no TLS.** The server exposes an unauthenticated HTTP API and UI on 9411. Every serious deployment fronts it with a reverse proxy handling TLS, authentication, and network policy — plan for this from day one.

**Sampling happens in the client.** The server stores everything it receives; there is no tail-based sampling or server-side rate control. Under traffic spikes, HTTP-reporting services can overwhelm collectors — Kafka transport exists largely as the backpressure buffer for this case.

**MySQL storage is a trap.** It reads as the easy option but queries degrade to multi-second latency at volume; the project itself labels it legacy[^4].

**Ecosystem direction.** New instrumentation effort industry-wide goes to OpenTelemetry, which can export to Zipkin format[^8]. If you deploy Zipkin today, assume your ingestion story will be OTel SDKs/collector emitting Zipkin-format spans, and weigh whether a natively OTLP backend serves you better long-term.

## When to Use / When Not

**Use when:**
- You want a tracing backend running in minutes — one jar, one port, embedded UI, no mandatory dependencies.
- Your stack already emits B3/Zipkin-format spans (Brave, Spring Cloud Sleuth-era JVM services, Envoy's Zipkin tracer).
- You need a lightweight, stable, self-hosted backend and your trace volume fits Cassandra/ES clusters you already operate.

**Avoid when:**
- You are starting greenfield on OpenTelemetry — a natively OTLP backend (Jaeger, Tempo, SigNoz) avoids format translation and gets active feature development.
- You need tail-based sampling, metrics correlation, or long cheap retention; Zipkin has none of these built in.
- You cannot operate Cassandra or Elasticsearch and need production-grade persistence anyway.

## Alternatives

- jaegertracing/jaeger — CNCF-graduated, natively OTLP, better Kubernetes story; use it instead for new OTel-based deployments.
- grafana/tempo — object-storage backend with cheap long retention and TraceQL; use it when trace volume and storage cost dominate.
- apache/skywalking — full APM (metrics + tracing + topology) rather than tracing alone; use it when you want one system for the whole picture.
- SigNoz/signoz — OTel-native all-in-one (traces, metrics, logs) on ClickHouse; use it as a self-hosted Datadog-shaped alternative.
- open-telemetry/opentelemetry-collector — not a storage backend, but replaces Zipkin's collector tier as the standard ingestion layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2012-06 | Open-sourced by Twitter; original Scala/Finagle implementation after Dapper[^1][^2]. |
| 1.x | 2015–2016 | Project moves to the OpenZipkin org; server rewritten in Java. |
| 2.0 | 2017 | v2 span model and simplified JSON API; v1 Thrift model becomes legacy. |
| 2.x | 2019 | Lens UI (React) replaces the classic UI. |
| 3.0 | 2024-01 | JRE 17 baseline; legacy modules and deprecated surface removed[^9]. |
| 3.x | 2024–2026 | Maintenance releases; OpenSearch coverage, dependency updates. |

## References

[^1]: Sigelman et al., "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure" — Google, 2010. https://research.google/pubs/pub36356/
[^2]: Twitter Engineering, "Distributed Systems Tracing with Zipkin" — 2012-06-07. https://blog.twitter.com/engineering/en_us/a/2012/distributed-systems-tracing-with-zipkin
[^3]: B3 propagation specification. https://github.com/openzipkin/b3-propagation
[^4]: Zipkin issue #1233 — MySQL storage performance. https://github.com/openzipkin/zipkin/issues/1233
[^5]: zipkin-dependencies — Spark job for aggregating dependency links. https://github.com/openzipkin/zipkin-dependencies
[^6]: zipkin-server documentation. https://github.com/openzipkin/zipkin/tree/master/zipkin-server
[^7]: W3C Trace Context. https://www.w3.org/TR/trace-context/
[^8]: OpenTelemetry Zipkin exporter specification. https://opentelemetry.io/docs/specs/otel/trace/sdk_exporters/zipkin/
[^9]: Zipkin 3.0.0 release. https://github.com/openzipkin/zipkin/releases/tag/3.0.0

## Tags

java, distributed-tracing, observability, tracing, microservices, apm, cassandra, elasticsearch, b3-propagation, self-hosted
