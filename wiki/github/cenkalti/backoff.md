# cenkalti/backoff

> Exponential-backoff retry primitives for Go — a small, single-purpose library that has been the community default for over a decade.

[GitHub repo](https://github.com/cenkalti/backoff) ·
[License: MIT](https://github.com/cenkalti/backoff/blob/v7/LICENSE)

## Overview

`backoff` is a Go implementation of the exponential-backoff algorithm, originally
ported from the `ExponentialBackOff` class in Google's HTTP Client Library for
Java[^1]. Its scope is deliberately narrow: compute the wait interval between
retries, and (optionally) drive a retry loop around a fallible operation. It does
not do circuit breaking, rate limiting, HTTP transport, or bulkheading — it does
one thing, and most of the Go ecosystem has at some point depended on it, directly
or transitively.

The core idea is small. A `BackOff` produces a growing, jittered sequence of
delays; a retry loop calls the operation, and on failure sleeps for the next delay
and tries again, until the operation succeeds, signals a permanent failure, or a
budget (attempt count, elapsed time, or context deadline) is exhausted. The
exponential growth with random jitter is what keeps a fleet of clients from
retrying in lockstep and hammering a recovering server (the "thundering herd").

The defining tension of this library is its API history. For roughly five years
`backoff/v4` was a stable, ubiquitous dependency with a plain
`Retry(op, backoff) error` signature. Starting with v5 (December 2024) the author
rewrote the public API around Go generics and a context-first, functional-options
style[^2]. The algorithm is unchanged, but the surface is not source-compatible
across the v4→v5 boundary, and because Go modules treat each major version as a
separate import path (`/v4` vs `/v7`), large codebases can end up linking several
majors at once through their transitive dependencies.

## Getting Started

```
go get github.com/cenkalti/backoff/v7
```

Note the `/v7` suffix — it is part of the import path, not just the tag.

```go
import "github.com/cenkalti/backoff/v7"

// Retry is generic over the operation's return type.
result, err := backoff.Retry(ctx, func() (string, error) {
	resp, err := http.Get("https://www.example.com")
	if err != nil {
		return "", err // transient — Retry will try again
	}
	defer resp.Body.Close()
	if resp.StatusCode >= 500 {
		return "", fmt.Errorf("server error: %s", resp.Status) // retried
	}
	if resp.StatusCode >= 400 {
		// client errors won't fix themselves — stop now.
		return "", backoff.Permanent(fmt.Errorf("client error: %s", resp.Status))
	}
	return "ok", nil
}, backoff.WithMaxTries(5))
```

## Architecture / How It Works

The library is built on one interface:

```go
type BackOff interface {
	NextBackOff() time.Duration // returns backoff.Stop (-1) to signal "give up"
	Reset()
}
```

Implementations include `ExponentialBackOff` (the default), `ConstantBackOff`,
and `ZeroBackOff`. `ExponentialBackOff` grows the interval by `Multiplier` each
call and randomizes it within `±RandomizationFactor` of the nominal value, capped
at `MaxInterval`. The stock defaults are `InitialInterval` 500ms,
`RandomizationFactor` 0.5 (so each delay lands in a ±50% window), `Multiplier`
1.5, and `MaxInterval` 60s[^3]. With those, delays climb roughly 0.5s → 0.75s →
1.1s → … and then flatten at 60s.

`Retry[T]` is the loop that consumes a `BackOff`. In v5+ it is
`Retry[T any](ctx context.Context, op Operation[T], opts ...RetryOption) (T, error)`.
Options tune behavior without mutating a shared struct: `WithBackOff`,
`WithMaxTries`, `WithMaxElapsedTime`, and `WithNotify` (a callback fired on each
failed attempt, useful for logging). `backoff.Permanent(err)` wraps an error so
the loop stops immediately rather than retrying a hopeless call.

Three stop conditions coexist and are reported distinctly. In v7, `Retry` returns
a structured `*RetryError` carrying the last operation error (`LastErr`) and the
reason it stopped (`Cause`), which you match with `errors.Is` against sentinels
`ErrPermanent`, `ErrMaxElapsedTime`, and `ErrExhausted`, or against
`context.Canceled` / `context.DeadlineExceeded`. `AsRetryError` reaches the struct
directly. This structured error surface is newer than the retry loop itself and is
the main ergonomic gain of the recent majors.

For callers who want the delays without the loop, `Ticker` exposes the same
sequence over a channel, letting you interleave backoff with `select` and your own
control flow.

## Production Notes

- **Two time limits with different semantics.** A context deadline is reactive: it
  interrupts the sleep between attempts and, if your operation honors the context,
  can abort an in-flight call — reported as `context.DeadlineExceeded`.
  `WithMaxElapsedTime` only bounds scheduling: it is checked between attempts,
  never interrupts a running operation, and reports `ErrMaxElapsedTime`.
- **`MaxElapsedTime` defaults to 15 minutes and is always on** unless you override
  it. A retry that "runs forever" often isn't — it quietly gave up at 15 minutes.
  Pass `backoff.WithMaxElapsedTime(0)` to disable it and rely solely on the
  context. In v4 this was a field on the `ExponentialBackOff` struct; in v5+ it
  moved to a `Retry` option, a subtle source of confusion when porting code.
- **`ExponentialBackOff` is stateful and not reset for you.** If you reuse a single
  instance across independent retry sessions, call `Reset()` (or construct a fresh
  one) or the second session starts mid-curve. The instance is also not safe for
  concurrent use by multiple goroutines.
- **Retrying non-idempotent operations is a footgun the library cannot solve.**
  `Retry` will happily re-issue a POST that already succeeded but whose response
  was lost. Gate retries on idempotency (or idempotency keys) yourself.
- **Major-version fan-out.** Because `/v4` and `/v7` are distinct import paths, a
  build can compile more than one copy of the package. This is harmless at runtime
  but inflates binaries and confuses grep-based audits; expect to pin and dedupe
  during upgrades.
- **Upgrading v4→v5+ is a rewrite, not a bump.** Call sites move from
  `Retry(op, b)` to `Retry(ctx, op, opts...)`, error inspection changes, and the
  operation signature becomes generic. There is no shim; migrate deliberately.

## When to Use / When Not

**Use when:**
- You need standard exponential backoff with jitter and don't want to hand-roll the
  interval math and stop conditions.
- You want a tiny, well-worn dependency with no transitive baggage.
- You need just the delay sequence (`BackOff` / `Ticker`) to feed your own loop.

**Avoid when:**
- You need broader resilience patterns — circuit breakers, bulkheads, deadlines,
  semaphores — in one place. This library is intentionally only backoff.
- You want HTTP-aware retries (per-status policy, `Retry-After` honoring,
  connection reuse) out of the box.
- You're on an older codebase pinned to v4 and don't want to absorb the v5+ API
  rewrite for a utility this small; vendoring a copy of `retry.go` is a legitimate
  alternative the README itself suggests.

## Alternatives

- avast/retry-go — declarative retry with a large option set; pick it when you want
  configurable retry-if predicates and per-attempt hooks without wiring a `BackOff`.
- sethvargo/go-retry — context-first, zero-dependency backoff constructors; use it
  when you prefer composing `Backoff` funcs (`WithJitter`, `WithCappedDuration`).
- jpillora/backoff — minimal duration generator with no retry loop; use it when all
  you need is "give me the next delay" and you own the loop.
- hashicorp/go-retryablehttp — an `http.Client` wrapper that retries transport-level
  and 5xx failures; use it when the thing you retry is specifically HTTP.
- eapache/go-resiliency — a grab-bag of patterns (breaker, deadline, retrier,
  semaphore); use it when backoff is one of several resilience concerns.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2016-03-21 | First tagged release; port of Google's Java `ExponentialBackOff`[^1]. |
| v2.0.0 | 2017-12-24 | Early API iteration. |
| v3.0.0 | 2019-05-11 | Continued refinement of the v1-style API. |
| v4.0.0 | 2019-12-26 | Go-modules major; the long-stable `Retry(op, b) error` API[^4]. |
| v4.3.0 | 2024-01-02 | Last v4 minor; generic `RetryWithData[T]` helpers added. |
| v5.0.0 | 2024-12-19 | Generics rewrite: context-first `Retry[T](ctx, op, opts...)`[^2]. |
| v6.0.0 | 2026-06-16 | Continued rework of the new API surface. |
| v7.0.0 | 2026-06-30 | Structured `*RetryError` with `LastErr` / `Cause` sentinels. |

## References

[^1]: README, "This is a Go port of the exponential backoff algorithm from Google's HTTP Client Library for Java." https://github.com/cenkalti/backoff#exponential-backoff
[^2]: v5.0.0 `retry.go` signature `Retry[T any](ctx context.Context, operation Operation[T], opts ...RetryOption) (T, error)`, tagged 2024-12-19. https://github.com/cenkalti/backoff/blob/v5.0.0/retry.go
[^3]: v7.0.0 `exponential.go` default constants (`DefaultInitialInterval` 500ms, `DefaultRandomizationFactor` 0.5, `DefaultMultiplier` 1.5, `DefaultMaxInterval` 60s). https://github.com/cenkalti/backoff/blob/v7/exponential.go
[^4]: v4.3.0 `retry.go` signature `Retry(o Operation, b BackOff) error`. https://github.com/cenkalti/backoff/blob/v4.3.0/retry.go

## Tags

go, golang, retry, exponential-backoff, resilience, jitter, http-client, error-handling, concurrency, library
