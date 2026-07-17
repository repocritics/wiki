# getsentry/sentry-go

> The official Go SDK for Sentry — error, panic, and performance instrumentation with an explicit Hub/Scope concurrency model.

[GitHub repo](https://github.com/getsentry/sentry-go) ·
[Official website](https://docs.sentry.io/platforms/go/) ·
[License: MIT](https://github.com/getsentry/sentry-go/blob/master/LICENSE)

## Overview

`sentry-go` is the official, first-party Go client for Sentry, the error-monitoring and application-performance product[^1]. It was introduced in 2019 as the successor to the older `raven-go` package, which is now legacy and no longer recommended for new code[^2]. The SDK captures unhandled panics, explicitly reported errors and messages, and — since the performance-monitoring work — distributed traces made of transactions and spans.

The SDK's defining design decision is its **Hub / Scope / Client** model, inherited from Sentry's cross-language SDK architecture rather than from Go idioms. A `Client` owns transport and options; a `Scope` holds contextual data (tags, user, breadcrumbs, extra); a `Hub` binds a client to a stack of scopes. The global `sentry.CaptureException` helpers operate on an implicit current hub. This layering is powerful but is also the source of the SDK's most common footgun: hubs are **not** safe to share across goroutines, so concurrent code must clone the hub or carry it through `context.Context`. Getting this wrong silently attaches the wrong user/request context to events.

The second defining tension is asynchronous delivery. The default transport buffers events and ships them on a background worker, so a program that exits (or a serverless handler that returns) before flushing will silently drop in-flight events. `defer sentry.Flush(timeout)` is mandatory boilerplate, not an optimization.

## Getting Started

```console
$ go get github.com/getsentry/sentry-go@latest
```

```go
package main

import (
	"log"
	"time"

	"github.com/getsentry/sentry-go"
)

func main() {
	err := sentry.Init(sentry.ClientOptions{
		Dsn:              "https://examplePublicKey@o0.ingest.sentry.io/0",
		Environment:      "production",
		Release:          "my-app@1.0.0",
		TracesSampleRate: 1.0, // omit or set 0 to disable tracing
	})
	if err != nil {
		log.Fatalf("sentry.Init: %s", err)
	}
	// Buffered events are sent asynchronously; flush before exit.
	defer sentry.Flush(2 * time.Second)

	sentry.CaptureMessage("it works")
}
```

DSN, release, and environment are read from `SENTRY_DSN`, `SENTRY_RELEASE`, and `SENTRY_ENVIRONMENT` if not set in code[^1].

## Architecture / How It Works

The runtime is built from four layers:

1. **Client** — created by `sentry.Init`. Holds `ClientOptions` (DSN, sample rates, `BeforeSend` hook, integrations) and a transport.
2. **Scope** — mutable bag of event data: tags, `User`, breadcrumbs, `Extra`, level, fingerprint. Scopes are pushed/popped so temporary context (`sentry.WithScope`) does not leak into sibling code.
3. **Hub** — a stack of scopes plus the active client. `sentry.CurrentHub()` is the process-global hub used by the top-level helper functions.
4. **Transport** — `HTTPTransport` (default) buffers events in a bounded queue and delivers them on a background goroutine; `HTTPSyncTransport` blocks until each event is sent, which is the correct choice for short-lived and serverless processes.

**Concurrency contract.** A hub must not be used from multiple goroutines simultaneously. The supported patterns are (a) `hub := sentry.CurrentHub().Clone()` per goroutine, or (b) propagate a hub via context with `sentry.SetHubOnContext` / `sentry.GetHubFromContext`[^3]. The framework integrations do this for you per request.

**Framework integrations** (`net/http`, gin, echo, fiber, fasthttp, iris, negroni) and logging bridges (logrus, slog, zerolog) live in the same repository but are published as **separate Go modules** with their own `go.mod` and version tags (for example `github.com/getsentry/sentry-go/gin`)[^1]. Each installs middleware that creates a request-scoped hub, recovers panics, and — when tracing is enabled — starts a transaction per request.

**Tracing** is built from `sentry.StartTransaction` and `span.StartChild`; sampling is governed by `TracesSampleRate` or a `TracesSampler` callback. Later releases added a profiler that samples goroutine stacks alongside transactions.

## Production Notes

- **Always flush.** In `main`, `defer sentry.Flush(2*time.Second)`. In AWS Lambda / Cloud Functions, either flush at the end of every invocation or use `HTTPSyncTransport` — the async transport will otherwise drop events when the runtime freezes the process.
- **Clone hubs across goroutines.** The single most common bug: spawning `go worker()` and calling `sentry.CaptureException` inside without cloning. Data races on the scope produce corrupted or cross-contaminated events. Use the per-request hub the middleware installs, and clone it before fanning out.
- **`CaptureException(nil)` is a no-op**, and reporting a wrapped error only gives good grouping/stack traces when the error carries a stack (e.g. `pkg/errors`) or you attach one; plain `fmt.Errorf` chains give a stack only from the capture site.
- **Sampling costs are per-transaction, not free.** `TracesSampleRate: 1.0` in high-throughput services generates large trace volume and Sentry quota consumption; sample down or use a dynamic `TracesSampler`.
- **`BeforeSend` is the scrubbing chokepoint.** PII (request bodies, headers, query strings) can flow into events via integrations; use `BeforeSend` / `BeforeSendTransaction` to redact before egress.
- **0.x versioning.** The SDK has remained on `0.x` for its entire life despite heavy production use; minor releases (`0.x.0`) can carry behavioral changes, so pin versions and read release notes rather than assuming semver-minor stability. Integration submodules are versioned in lockstep with the core.
- **Two-release Go support window.** The project supports only the two most recent Go major releases, matching Go's own policy[^1]; older toolchains are not guaranteed to build.

## When to Use / When Not

**Use when:**
- You are already on Sentry and want first-party error grouping, breadcrumbs, releases, and panic recovery in Go.
- You want error monitoring and tracing from one SDK with ready-made middleware for a common web framework.
- You need managed, searchable issue triage rather than raw logs/metrics.

**Avoid when:**
- You want vendor-neutral telemetry you can point at any backend — OpenTelemetry is the portable choice.
- You only need metrics/logs, not error-issue workflows — a logging or metrics stack is lighter.
- You cannot send data to a hosted service and do not want to run self-hosted Sentry, which is a substantial infrastructure commitment.

## Alternatives

- open-telemetry/opentelemetry-go — use instead when you want vendor-neutral tracing/metrics that can target Sentry, Jaeger, or any OTLP backend rather than Sentry-specific APIs.
- getsentry/raven-go — the deprecated predecessor; use only to keep a legacy codebase running until you migrate to `sentry-go`.
- bugsnag/bugsnag-go — use when your organization standardizes on Bugsnag for error monitoring instead of Sentry.
- rollbar/rollbar-go — use when Rollbar is your error-tracking vendor.
- uptrace/uptrace-go — use when you want an open-source, OpenTelemetry-based APM you can self-host without Sentry's stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2019 | Initial release; successor to `raven-go`, Hub/Scope/Client model[^2]. |
| 0.x (perf) | ~2022 | Performance monitoring: transactions, spans, `TracesSampleRate`. |
| 0.x (profiling) | ~2023 | Goroutine profiling alongside tracing. |
| 0.x (modularized) | ongoing | Framework/logging integrations split into separate Go modules. |

## References

[^1]: Sentry Go SDK documentation and repository README. https://docs.sentry.io/platforms/go/
[^2]: Migration from the legacy `raven-go` client. https://docs.sentry.io/platforms/go/migration/
[^3]: Concurrency guidance — hubs and scopes across goroutines. https://docs.sentry.io/platforms/go/concurrency/

## Tags

go, golang, error-monitoring, crash-reporting, observability, tracing, sdk, sentry, apm, performance-monitoring
