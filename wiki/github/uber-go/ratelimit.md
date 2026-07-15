# uber-go/ratelimit

> A blocking leaky-bucket rate limiter for Go: call `Take()` before each operation and it sleeps until you're allowed to proceed.

[GitHub repo](https://github.com/uber-go/ratelimit) ·
[Docs](https://pkg.go.dev/go.uber.org/ratelimit) ·
[License: MIT](https://github.com/uber-go/ratelimit/blob/main/LICENSE)

## Overview

`uber-go/ratelimit` is a small, single-purpose library that throttles a stream
of operations to a fixed rate — for example, capping outbound calls to a
downstream API or database at N per second. It implements the leaky-bucket
algorithm, but instead of a background ticker refilling a bucket at discrete
intervals, it computes the earliest permissible time for the next operation
from the elapsed time since the previous one, then blocks (`time.Sleep`) until
that moment. The whole public surface is essentially one method: `Take()`.

The defining design choice is that it is *blocking by default and only
blocking*. There is no `Allow()`-style boolean check, no reservation object,
and — importantly — no `context.Context` parameter. This makes the API almost
impossible to misuse for its intended job (smoothing a caller's own request
rate), but it also means the limiter cannot be cancelled or given a deadline
once a goroutine is parked inside `Take()`. The maintainers state the goal
explicitly: a simple API and minimal overhead, with `golang.org/x/time/rate`
recommended for anything more complex[^1].

Import path is `go.uber.org/ratelimit` (not the GitHub path). It is part of
Uber's `go.uber.org` family alongside `zap` and `fx`, and is widely vendored
but not heavily featured — the repo has seen infrequent commits in recent years
(last push 2024-05-01), which for a library this stable is expected rather than
alarming.

## Getting Started

```bash
go get go.uber.org/ratelimit
```

```go
package main

import (
	"fmt"
	"time"

	"go.uber.org/ratelimit"
)

func main() {
	rl := ratelimit.New(100) // 100 operations per second

	prev := time.Now()
	for i := 0; i < 10; i++ {
		now := rl.Take() // blocks until the next slot; returns the time it unblocked
		fmt.Println(i, now.Sub(prev))
		prev = now
	}
	// After the first call, each iteration is spaced ~10ms apart.
}
```

Rates other than per-second use the `Per` option, e.g.
`ratelimit.New(20, ratelimit.Per(time.Minute))` for 20/min.

## Architecture / How It Works

`Take()` records the timestamp of the last permitted operation and the target
interval (`1 / rate`). On each call it computes when the next operation is
allowed, sleeps until then if necessary, and returns the unblock time. Because
timing is derived from wall-clock deltas rather than a background goroutine,
there is no ticker to leak and idle limiters cost nothing.

Two internal limiter implementations exist, selected by build/options; a
mutex-guarded version and an atomic (CAS-based) version, the latter being the
default in current releases to reduce contention under many concurrent callers.
The `Clock` interface abstracts `time.Now`/`time.Sleep`, which is what makes the
limiter unit-testable with a fake clock instead of real sleeps.

The subtle behavioural detail is **slack**. By default the limiter allows a
bounded amount of accumulated "credit" (default slack of 10) so that a caller
which has been idle can briefly burst to catch up, rather than being pinned to
exact even spacing. This smooths real workloads but surprises anyone expecting
strictly uniform intervals. `WithoutSlack` disables it for hard, evenly-spaced
pacing; `WithSlack(n)` tunes the amount.

## Production Notes

- **No cancellation or deadline.** `Take()` takes no `context.Context`. A
  goroutine blocked inside it will sleep for the full computed duration even if
  the surrounding request is cancelled. If you need context-aware waiting, this
  is the wrong library — reach for `x/time/rate`'s `Wait(ctx)`.
- **Single-process only.** The limiter is in-memory state on one instance. It
  does not coordinate across processes or hosts; for distributed limiting you
  need a Redis/GCRA-backed solution.
- **One global stream, not per-key.** A limiter instance throttles everything
  that calls it, together. Per-client / per-key limiting requires you to
  maintain your own map of limiters and handle eviction yourself.
- **Sleep granularity caps effective rate.** Because pacing is `time.Sleep`,
  OS timer resolution (~1ms on many platforms) makes very high rates (tens of
  thousands/sec) imprecise. It is designed for the low-thousands-per-second
  range and below, not as a hot-path token counter.
- **Windows timer precision.** The example test can fail on Windows due to
  known Go timer-precision issues; the maintainers explicitly do not work
  around it[^2].
- **Slack is on by default.** If you assume `Take()` guarantees minimum
  spacing between every pair of calls, test your assumption — the default
  slack of 10 permits short bursts after idle periods.

## When to Use / When Not

**Use when:**
- You control the caller and want to cap your own outbound rate to a service.
- Blocking backpressure is acceptable (or desirable) as flow control.
- You want a one-line API and don't need tuning knobs.
- Everything runs in a single process.

**Avoid when:**
- You need a non-blocking check (`Allow()`) or reservations.
- You need `context` cancellation / timeouts while waiting.
- You need distributed or per-key/per-tenant limiting.
- You're gating an extremely high-frequency hot path where sleep granularity
  and blocking overhead matter.

## Alternatives

- golang/time (`golang.org/x/time/rate`) — the standard token-bucket limiter;
  use it when you need `Allow`/`Reserve`/`Wait(ctx)`, bursts, or context cancellation.
- juju/ratelimit — classic token-bucket implementation; use it when you want
  explicit token/burst semantics rather than blocking-only pacing.
- throttled/throttled — GCRA-based limiting with pluggable stores; use it for
  HTTP request limiting keyed by client with a memory or Redis backend.
- ulule/limiter — HTTP middleware with memory/Redis stores; use it when you want
  drop-in per-IP/per-route limiting across instances.
- go-redis/redis_rate — Redis GCRA limiter; use it when the limit must be shared
  across many processes or hosts.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-09 | Repo created; blocking leaky-bucket with elapsed-time refill[^1]. |
| v0.1.0 | 2017 | First tagged release. |
| v0.2.0 | 2021 | Atomic/CAS limiter, `Per` and slack options (`WithSlack`/`WithoutSlack`). |
| v0.3.0 | 2023 | Default atomic limiter refinements; latest tagged line. |

Exact release dates for tagged versions should be confirmed against the repo's
releases page; the year-level dates above reflect the general timeline.

## References

[^1]: README and FAQ, uber-go/ratelimit — leaky-bucket design and the stated
"simple API / minimal overhead" goal, with `x/time/rate` recommended for complex
cases. https://github.com/uber-go/ratelimit
[^2]: uber-go/ratelimit FAQ on Windows timer precision (references golang/go#44343).
https://github.com/uber-go/ratelimit#faq

## Tags

go, rate-limiting, leaky-bucket, concurrency, throttling, backpressure, uber, library
