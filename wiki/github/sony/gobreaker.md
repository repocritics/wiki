# sony/gobreaker

> A single-file, dependency-free Circuit Breaker for Go — a state machine that stops calling a failing dependency until it looks healthy again.

[GitHub repo](https://github.com/sony/gobreaker) ·
[License: MIT](https://github.com/sony/gobreaker/blob/master/LICENSE)

## Overview

gobreaker is Sony's implementation of the Circuit Breaker pattern as described in Michael Nygard's *Release It!* and Microsoft's Azure architecture guidance[^1]. It is deliberately small: the core is one Go file, standard library only, and the public surface is essentially one constructor, one `Execute` method, and a `Settings` struct. Since 2015 it has become one of the most-cited circuit breaker libraries in the Go ecosystem, and is frequently the reference other Go breaker libraries measure themselves against.

The value proposition is narrow and honest. gobreaker does not do retries, timeouts, bulkheads, rate limiting, or fallbacks — it only decides whether a call should be attempted at all, based on the recent success/failure history of the thing you are calling. When a downstream service starts failing, the breaker "opens" and fails fast (returning `ErrOpenState`) instead of piling load onto a struggling dependency; after a cooldown it probes with a limited number of requests before fully closing again.

The defining tension is scope discipline versus completeness. Because gobreaker refuses to be a general resilience toolkit, wiring it into a real service means you supply the surrounding retry/timeout/fallback logic yourself. Teams wanting an all-in-one resilience library reach for something larger; teams wanting exactly one well-understood primitive keep coming back to gobreaker. The v2 line added Go generics so `Execute` is typed rather than returning `interface{}`[^2].

## Getting Started

```
go get github.com/sony/gobreaker/v2
```

```go
package main

import (
	"io"
	"net/http"
	"time"

	"github.com/sony/gobreaker/v2"
)

var cb = gobreaker.NewCircuitBreaker[[]byte](gobreaker.Settings{
	Name:        "http-get",
	MaxRequests: 3,               // probes allowed while half-open
	Timeout:     30 * time.Second, // how long to stay open before probing
	ReadyToTrip: func(c gobreaker.Counts) bool {
		return c.ConsecutiveFailures > 5
	},
})

func Get(url string) ([]byte, error) {
	return cb.Execute(func() ([]byte, error) {
		resp, err := http.Get(url)
		if err != nil {
			return nil, err
		}
		defer resp.Body.Close()
		return io.ReadAll(resp.Body)
	})
}
```

`Execute` returns `ErrOpenState` (or `ErrTooManyRequests` in half-open) instantly when the breaker is not accepting calls; otherwise it runs the closure and records the outcome.

## Architecture / How It Works

gobreaker is a three-state machine: **closed** (calls pass through, outcomes counted), **open** (calls rejected immediately), and **half-open** (a limited number of probe calls allowed). Transitions are driven entirely by a `Counts` struct tracking requests, total and consecutive successes/failures, and — in v2 — exclusions.

The trip decision is fully delegated to your `ReadyToTrip(Counts) bool`. The default trips after 5 consecutive failures, but the common production configuration uses a failure *ratio* over a minimum request volume, so a low-traffic breaker does not flap on one or two errors. `IsSuccessful(err)` lets you classify which errors count as failures (e.g. treat HTTP 4xx as success), and `IsExcluded(err)` lets you drop errors like `context.Canceled` from the counts entirely.

Counting is bucketed by time. `Interval` defines how often the closed-state counts are cleared; a request that succeeds resets consecutive-failure counts. The v2 line added a **rolling window** via `BucketPeriod`: instead of a single fixed window that clears all at once, counts age out gradually per bucket, which avoids the "cliff" where a breaker forgets all recent history at an interval boundary. If `BucketPeriod <= 0` it falls back to the classic fixed-window behavior, so existing configs are unaffected.

Two consumption shapes exist. `CircuitBreaker.Execute(fn)` wraps a function. `TwoStepCircuitBreaker` exposes `Allow() (done func(success bool), err error)` for cases where you cannot express the call as a single closure — you ask permission, do the work, then report the outcome yourself. Concurrency is handled with a single mutex around state and counts; there is no per-call goroutine or channel machinery, which keeps overhead low but means the lock is a shared point for very high-throughput callers.

## Production Notes

- **The default `ReadyToTrip` is rarely what you want.** Five consecutive failures ignores volume: a nearly-idle endpoint and a firehose are treated identically. Most real deployments switch to a ratio-with-minimum-requests predicate.
- **One breaker per dependency, not per process.** The breaker's counts are global to that instance. Sharing a single breaker across unrelated downstreams means one bad dependency trips calls to healthy ones. Create a distinct `CircuitBreaker` (distinct `Name`) per remote target.
- **State is per-process, not shared across replicas.** Each pod/instance maintains its own breaker state, so twenty replicas will each independently discover a downstream outage. This is usually acceptable, but it means a global "the dependency is down" view is not built in. v2 introduced storage-backed distributed state as an option; if you rely on it, verify its current API against the repo rather than this page.
- **Panics propagate.** If the wrapped function panics, gobreaker records it as a failure and re-panics. Your calling code still needs its own recovery if a panic must not crash the goroutine.
- **`MaxRequests` gates recovery, not steady state.** It only limits concurrent probes while half-open. Setting it to 0 means exactly one probe request is allowed at a time — a slow way to recover a high-traffic path.
- **v1 → v2 is a breaking import change.** The module path is `github.com/sony/gobreaker/v2` and `Execute` is now generic (`CircuitBreaker[T]`). Migrating means updating the import path and adding type parameters; the v1 tag remains importable for code that cannot move.
- **No built-in metrics.** Observability is via the `OnStateChange` callback; you export open/close transitions to your own metrics system. There is no Prometheus collector in the box.

## When to Use / When Not

**Use when:**
- You need exactly the circuit breaker primitive and want to keep dependencies and surface area minimal.
- You want full control over the trip predicate and error classification.
- You are wrapping a small number of well-identified remote dependencies.

**Avoid when:**
- You want retries, timeouts, hedging, bulkheads, and fallbacks in one package — gobreaker is only the breaker.
- You need cluster-wide shared breaker state as a first-class, batteries-included feature.
- You want built-in metrics/dashboards without writing the export glue yourself.

## Alternatives

- afex/hystrix-go — Netflix Hystrix port with bulkheads and metrics; use it when you want the fuller Hystrix model, accepting it is far less actively maintained.
- failsafe-go/failsafe-go — composable resilience policies (retry, timeout, circuit breaker, hedge); use it when the breaker is one of several policies you want to stack.
- eapache/go-resiliency — small toolkit whose `breaker` package is a lighter alternative; use it when you also want its retrier/deadline helpers from one dependency.
- mercari/go-circuitbreaker — context-aware breaker with a similar philosophy; use it when first-class `context.Context` integration matters to you.
- slok/goresilience — middleware-style runners for multiple resilience patterns; use it when you prefer composing patterns as chained runners.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-05-29 | First release; classic closed/open/half-open breaker, fixed-window counts[^1]. |
| v2.0 | 2024 | Generics (`CircuitBreaker[T]`, Go 1.18+), new `/v2` module path[^2]. |
| v2.x | 2026-02 | Rolling-window counting via `BucketPeriod`; `IsExcluded` and `TotalExclusions`. |

## References

[^1]: Microsoft, "Circuit Breaker pattern" — Azure Architecture Center; and Michael Nygard, *Release It!* (2007), the pattern's canonical description. https://learn.microsoft.com/azure/architecture/patterns/circuit-breaker
[^2]: gobreaker v2 module documentation. https://pkg.go.dev/github.com/sony/gobreaker/v2

## Tags

go, golang, circuit-breaker, resilience, fault-tolerance, microservices, state-machine, reliability, library, mit-license
