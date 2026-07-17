# open-telemetry/opentelemetry-go

> The Go implementation of the OpenTelemetry API and SDK — vendor-neutral traces, metrics, and logs.

[GitHub repo](https://github.com/open-telemetry/opentelemetry-go) ·
[Official website](https://opentelemetry.io/docs/languages/go) ·
[License: Apache-2.0](https://github.com/open-telemetry/opentelemetry-go/blob/main/LICENSE)

## Overview

OpenTelemetry-Go is the reference Go implementation of OpenTelemetry, the CNCF
observability standard formed in 2019 from the merger of OpenTracing and
OpenCensus[^1]. It provides two distinct things in one repository: a stable,
dependency-light **API** that instrumentation code calls, and a pluggable
**SDK** that applications wire up at startup to actually collect, batch, and
export telemetry. The split is the project's defining architectural decision —
libraries depend only on the API, so importing an instrumented library adds no
telemetry cost unless the application installs an SDK.

The signals mature independently. Tracing has been stable since v1.0.0 (2021);
metrics stabilized later (v1.16.0, 2023) after a significant API redesign; logs
remain beta as of 2026[^2]. This staggered stability is a frequent source of
confusion, because all three share the same repo and overlapping
`go.opentelemetry.io/otel/*` module tree but carry different guarantees.

This repo is also deliberately small. The API, SDK, and core exporters (OTLP,
Prometheus, stdout, Zipkin) live here; instrumentation libraries (`otelhttp`,
`otelgin`, database wrappers) and most other exporters live in the separate
`opentelemetry-go-contrib` repo[^3]. New users routinely look for `otelhttp`
here and do not find it.

## Getting Started

Modules use the `go.opentelemetry.io/otel` vanity import path, not a
`github.com/...` path:

```bash
go get go.opentelemetry.io/otel \
       go.opentelemetry.io/otel/sdk \
       go.opentelemetry.io/otel/exporters/stdout/stdouttrace
```

```go
package main

import (
	"context"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/sdk/trace"
)

func main() {
	exp, err := stdouttrace.New()
	if err != nil {
		panic(err)
	}
	tp := trace.NewTracerProvider(trace.WithBatcher(exp))
	defer tp.Shutdown(context.Background()) // flush pending spans

	otel.SetTracerProvider(tp) // register globally

	tracer := otel.Tracer("example")
	_, span := tracer.Start(context.Background(), "do-work")
	defer span.End()
}
```

Without the `SetTracerProvider` call, `otel.Tracer(...)` returns a no-op
tracer and produces no output — silently. This is by design (libraries stay
cheap) but is the most common "why is there no data" footgun.

## Architecture / How It Works

The dependency graph is layered:

1. **API** (`go.opentelemetry.io/otel`, `.../trace`, `.../metric`, `.../log`) —
   interfaces plus a global registry. Calling code obtains a `Tracer`, `Meter`,
   or `Logger` and records against a `context.Context`. Default impls are no-ops.
2. **SDK** (`.../sdk/...`) — the real providers. A `TracerProvider` owns
   samplers, span processors, resource attributes, and exporters; a
   `MeterProvider` owns readers and aggregation temporality.
3. **Exporters** — translate SDK data to a wire format. OTLP (gRPC or HTTP) is
   the recommended path to a Collector or backend.

Context propagation is explicit: spans chain through `context.Context`, and
cross-process propagation uses a configured `TextMapPropagator` (W3C Trace
Context by default, but you must set it via `otel.SetTextMapPropagator`).

Sampling is **head-based** in the SDK — decided when a span starts. Tail-based
sampling (deciding after seeing a full trace) requires the OpenTelemetry
Collector downstream. **Semantic conventions** (`semconv`) are versioned
packages of standard attribute keys (`http.request.method`, `service.name`);
they are pinned per release and change between versions, a real churn source
for dashboards and alerts.

## Production Notes

**Module version skew is the top operational hazard.** The repo publishes many
separately-versioned modules under one path. The stable `otel`, `sdk`, and
exporter modules track a `v1.x` line; some pieces (metrics historically, logs
now) sit on `v0.x`. Mixing incompatible versions produces confusing build or
runtime errors. Keep all `go.opentelemetry.io/otel/*` requires bumped together,
and pin `opentelemetry-go-contrib` to a release compatible with your core
version.

**Metric cardinality is unbounded by default.** Every unique attribute-set on a
counter/histogram creates a distinct time series held in memory. High-
cardinality attributes (user IDs, request paths with IDs, full URLs) cause
memory growth and backend cost blowups. Use views to drop or bucket attributes.

**Batch vs simple span processors.** `WithBatcher` (async, batched) is correct
for production; the simple/syncer processor blocks on every span and is for
tests only. Always call `TracerProvider.Shutdown` (and `MeterProvider.Shutdown`)
on exit or you will drop the last, unflushed batch — easy to miss in short-lived
jobs and serverless handlers.

**OTLP exporter config.** The exporter reads standard `OTEL_EXPORTER_OTLP_*`
env vars, and gRPC defaults to insecure. Endpoint, TLS, and header settings via
env vs. code options overlap and can conflict; be explicit.

**Logs are beta.** The log bridge API/SDK works but its surface and stability
guarantees are still evolving; avoid deep coupling in long-lived code.

**Go version policy.** The project follows upstream Go's support window and
drops compatibility testing for Go versions once they age out, so staying on an
EOL Go toolchain will eventually strand you on an old otel release.

## When to Use / When Not

**Use when:**
- You want vendor-neutral instrumentation that can target any OTLP backend
  (Jaeger, Tempo, Datadog, Honeycomb, Grafana Cloud) without code changes.
- You need traces, metrics, and logs under one correlated context model.
- You are building a library and want to emit telemetry without forcing a
  backend or SDK cost on consumers (depend on the API only).

**Avoid when:**
- You only need Prometheus metrics scraped from one service —
  `prometheus/client_golang` is simpler and has less surface area.
- You are locked into a single APM vendor and want that vendor's richer
  auto-instrumentation and ergonomics (their native agent may cover more).
- You need tail-based sampling or heavy pipeline processing but do not want to
  run a Collector — the in-process SDK cannot do it.

## Alternatives

- open-telemetry/opentelemetry-collector — not a replacement but the usual
  downstream companion; use it when you need vendor-agnostic routing,
  processing, or tail-based sampling outside the app.
- DataDog/dd-trace-go — use when you are all-in on Datadog and want their
  native APM instrumentation and product integration over neutrality.
- prometheus/client_golang — use when you only need metrics exposed for
  Prometheus scraping and do not want tracing or an export pipeline.
- census-instrumentation/opencensus-go — the deprecated predecessor; only
  relevant for migrating legacy OpenCensus code (bridge shims exist).

## History

| Version | Date | Notes |
|---------|------|-------|
| pre-1.0 | 2019–2021 | Formed from OpenTracing + OpenCensus merger; frequent API churn[^1]. |
| v1.0.0 | 2021-09 | Tracing API and SDK declared stable[^2]. |
| v1.11.x | 2022 | Metrics API redesign ahead of stabilization. |
| v1.16.0 | 2023-05 | Metrics API and SDK declared stable. |
| v1.x + logs beta | 2024–2026 | Log bridge API/SDK added and iterated (beta)[^2]. |

## References

[^1]: OpenTelemetry — history and OpenTracing/OpenCensus merger. https://opentelemetry.io/docs/what-is-opentelemetry/
[^2]: OpenTelemetry-Go project status (Traces/Metrics stable, Logs beta) and versioning guarantees. https://github.com/open-telemetry/opentelemetry-go/blob/main/VERSIONING.md
[^3]: OpenTelemetry-Go Contrib — instrumentation libraries and additional exporters. https://github.com/open-telemetry/opentelemetry-go-contrib

## Tags

go, observability, opentelemetry, tracing, metrics, logging, otlp, telemetry, distributed-tracing, cncf, instrumentation, sdk
