# DataDog/dd-trace-go

> Datadog's official Go client library for APM tracing, continuous profiling, and application security — data flows through the local Datadog Agent, not straight to Datadog.

[GitHub repo](https://github.com/DataDog/dd-trace-go) ·
[Official docs](https://docs.datadoghq.com/tracing/) ·
[License: Apache-2.0 OR BSD-3-Clause](https://github.com/DataDog/dd-trace-go/blob/main/LICENSE)

## Overview

dd-trace-go is the Go SDK for Datadog's observability suite. A single repository ships three product surfaces: APM distributed tracing (`ddtrace/tracer`), the continuous profiler (`profiler`), and Application Security Management / AppSec, which is not a standalone package but rides inside the tracer and is toggled with `DD_APPSEC_ENABLED=true`[^1]. First tagged in 2016, it is a vendor-specific agent: it exists to get spans, profiles, and security signals into Datadog, and its design assumes you are a paying Datadog customer.

The defining architectural fact is indirection through the Datadog Agent. The library does not POST traces to Datadog's public API; it ships them to a local Agent process (default `localhost:8126`), which buffers, samples, and forwards them[^2]. This decouples your app's lifecycle from network calls to Datadog but adds a mandatory piece of deployment infrastructure — no Agent, no traces. Teams evaluating dd-trace-go against agentless SaaS tracers should weigh that operational cost up front.

The other defining tension is the v1-to-v2 split. In June 2025 the project shipped v2.0.0, a structural rewrite that moved the module to `github.com/DataDog/dd-trace-go/v2` and broke every `contrib` integration out into independently versioned nested modules[^3]. The v1 line (`v1.7x.x`) is still receiving releases in parallel, so at any given moment two supported major versions coexist with different import paths and different module topologies.

## Getting Started

```bash
go get github.com/DataDog/dd-trace-go/v2/ddtrace/tracer
go get github.com/DataDog/dd-trace-go/v2/profiler
# integrations are separate nested modules:
go get github.com/DataDog/dd-trace-go/contrib/net/http/v2
```

```go
package main

import (
	"net/http"

	"github.com/DataDog/dd-trace-go/v2/ddtrace/tracer"
)

func main() {
	tracer.Start(
		tracer.WithService("web-api"),
		tracer.WithEnv("production"),
	)
	defer tracer.Stop() // flushes buffered spans on shutdown

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		span, ctx := tracer.StartSpanFromContext(r.Context(), "handler.root")
		defer span.Finish()
		_ = ctx // propagate ctx into downstream calls / DB queries
		w.Write([]byte("ok"))
	})
	http.ListenAndServe(":8080", nil)
}
```

The tracer is a process-global singleton: `tracer.Start()` must run once at startup before any span is created, and `tracer.Stop()` must run at shutdown or in-flight spans are dropped. Most real applications wrap frameworks with a `contrib` package rather than hand-instrumenting every handler.

## Architecture / How It Works

The core is `ddtrace/tracer`: spans are created from a `context.Context`, propagation happens by threading that context through call chains, and finished spans are batched and flushed to the Agent asynchronously on a background goroutine. Sampling is head-based by default — the decision to keep a trace is made at the root and propagated — with configurable rates and rule-based overrides.

`contrib` is where most of the surface area lives. Each package wraps a popular library — `net/http`, `gorilla/mux`, `google.golang.org/grpc`, `database/sql`, Redis clients, Kafka clients, and dozens more — so instrumentation is opt-in per dependency. Since v2 each of these is its own Go module with its own version tag, published under paths like `.../contrib/gorilla/mux/v2`.

For teams that don't want to hand-wire every integration, Datadog offers Orchestrion, a compile-time auto-instrumentation tool that rewrites your build via Go's `-toolexec` hook to inject `contrib` calls automatically[^4]. It is a separate tool but tightly co-developed with this repo.

The `profiler` package runs independently of tracing: it periodically collects CPU, heap, goroutine, mutex, and block profiles (Go's standard pprof profiles) and ships them — again through the Agent — to Datadog. There is also an OpenTelemetry bridge so code written against the OTel Go API can emit through Datadog's tracer, though the native `ddtrace` API remains the first-class path.

## Production Notes

**The Agent is a hard dependency.** Traces and profiles go to a Datadog Agent (`localhost:8126` for traces by default). In Kubernetes this typically means a node-level Agent DaemonSet and configuring `DD_AGENT_HOST`. If the Agent is unreachable, spans are buffered and eventually dropped — the app keeps running but you silently lose data.

**Nested contrib modules cause version skew.** The v2 restructuring means your `go.mod` can pin `ddtrace/tracer` at one v2 minor and a `contrib` package at another. They are released in lockstep by Datadog but resolved independently by `go mod`, so mismatches are easy to introduce and `go mod tidy` can pull surprising combinations. Keep tracer and contrib versions aligned.

**v1 → v2 migration is non-trivial.** It is not a find-and-replace: import paths change to `/v2`, contrib packages move to nested modules, and some APIs changed shape. Datadog maintains a dedicated `MIGRATING.md`[^5]. Because v1 is still supported, there is no forcing function — many codebases run v1 indefinitely, which is viable but leaves them off the current feature line.

**Overhead is real but bounded.** Tracing adds per-span allocation and a background flush goroutine; the usual mitigations are sampling and not over-instrumenting hot paths. The profiler's overhead is low but non-zero and should be validated under load rather than assumed free.

**Go version policy is strict.** The library supports only the two latest Go releases, matching Go's own support window, and only first-class ports[^1]. Staying on an older Go toolchain will eventually strand you on an old tracer.

**AppSec is a mode, not a module.** Enabling `DD_APPSEC_ENABLED=true` activates in-tracer security monitoring (SSRF, SQLi, XSS detection). It shares the tracer's lifecycle, so its coverage is only as good as your `contrib` instrumentation coverage.

## When to Use / When Not

**Use when:**
- You are a Datadog customer and want APM, profiling, and AppSec from one Go SDK.
- You already run Datadog Agents and want deep, out-of-the-box instrumentation for common Go libraries.
- You want compile-time auto-instrumentation (via Orchestrion) rather than editing every call site.

**Avoid when:**
- You want a vendor-neutral pipeline: OpenTelemetry lets you switch backends without re-instrumenting.
- You can't or won't run a Datadog Agent alongside your app.
- You only need profiling or only need tracing and don't want to adopt a vendor SDK for it.

## Alternatives

- open-telemetry/opentelemetry-go — use when you want a vendor-neutral tracing standard and the freedom to change observability backends later (Datadog can ingest OTLP).
- DataDog/orchestrion — companion, not competitor: use when you want dd-trace-go instrumentation injected at compile time instead of by hand.
- grafana/pyroscope — use when continuous profiling is all you need and you'd rather run open-source, self-hosted profiling.
- newrelic/go-agent — use when your APM vendor is New Relic instead of Datadog.
- open-telemetry/opentelemetry-go-contrib — use when you've chosen OTel and need per-library instrumentation equivalents to Datadog's `contrib`.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2016-09-08 | Initial Go tracer development begins[^6]. |
| v1.0.0 | 2018-06-06 | First stable v1 release[^6]. |
| v1.7x.x | 2025 | v1 line still actively released in parallel with v2. |
| v2.0.0 | 2025-06-05 | Major restructure: `/v2` module path, contrib split into nested modules[^3]. |
| v2.9.1 | 2026-06-26 | Recent v2 patch release[^6]. |

## References

[^1]: DataDog/dd-trace-go README — packages, AppSec enablement, and Go support policy. https://github.com/DataDog/dd-trace-go
[^2]: Datadog docs, "Tracing Go Applications" — the tracer sends spans to the Datadog Agent (default `localhost:8126`). https://docs.datadoghq.com/tracing/trace_collection/library_config/go/
[^3]: dd-trace-go v2.0.0 release (2025-06-05) — module path `github.com/DataDog/dd-trace-go/v2`, nested contrib modules. https://github.com/DataDog/dd-trace-go/releases/tag/v2.0.0
[^4]: DataDog/orchestrion — compile-time auto-instrumentation via Go `-toolexec`. https://github.com/DataDog/orchestrion
[^5]: dd-trace-go migration guide (v1 to v2). https://github.com/DataDog/dd-trace-go/blob/main/MIGRATING.md
[^6]: GitHub repository metadata and release tags, retrieved 2026-07-17. https://github.com/DataDog/dd-trace-go/releases

## Tags

go, apm, distributed-tracing, observability, profiling, appsec, datadog, opentelemetry, instrumentation, agent-based, monitoring
