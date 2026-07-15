# uber-go/goleak

> A goroutine leak detector for Go tests — assert that a test or package left no stray goroutines running.

[GitHub repo](https://github.com/uber-go/goleak) ·
[Docs](https://pkg.go.dev/go.uber.org/goleak) ·
[License: MIT](https://github.com/uber-go/goleak/blob/master/LICENSE)

## Overview

`goleak` is a small testing library from Uber's Go group that fails a test if
goroutines are still running when it finishes. Its premise is that a goroutine
left alive after a test is usually a bug — a background loop that was never
signalled to stop, an HTTP client whose idle connections were never closed, a
`time.Ticker` never stopped, a channel reader blocked forever. These leaks
rarely break the test that spawned them; they accumulate and surface later as
memory growth or flaky, order-dependent failures in CI.

The library is deliberately narrow. It does not trace ownership, attribute a
leaked goroutine to the code that started it, or run in production. It snapshots
the current set of goroutines by parsing the runtime stack dump, filters out
ones it considers benign, and if anything remains it retries for a short window
before failing with the offending stacks printed.

The defining tension: goroutine liveness at the moment a test returns is
inherently racy — a goroutine may be legitimately mid-shutdown when `VerifyNone`
runs. goleak papers over this with bounded retries and a default ignore-list,
which makes it reliable in practice but means it is a heuristic, not a proof.

## Getting Started

```bash
go get -u go.uber.org/goleak
```

```go
// Per-test: assert no leaks when this test returns.
func TestWorker(t *testing.T) {
	defer goleak.VerifyNone(t)

	w := NewWorker()
	w.Start()
	w.Stop() // if Stop() forgets to join its goroutine, this test fails.
}
```

```go
// Per-package: one check after the whole package's tests run.
func TestMain(m *testing.M) {
	goleak.VerifyTestMain(m)
}
```

`VerifyTestMain` is the cheaper, more common setup: it runs the leak check once
rather than after every test. The tradeoff is attribution — a failure tells you
the package leaked, not which test did it.

## Architecture / How It Works

goleak has essentially no runtime machinery. The core is `stack.All()`, which
calls `runtime.Stack` to capture every goroutine's stack trace as text and
parses each into a `Stack` with an ID, state, and function. `VerifyNone` takes
that snapshot, drops any goroutine matching a filter, and if the remainder is
non-empty, sleeps and retries — by default up to ~20 attempts with exponential
backoff totalling a fraction of a second — before reporting failure. The retry
loop exists precisely because goroutines spawned by the test may still be
unwinding when the check fires.

The default filter is where most real-world behavior lives. Out of the box
goleak ignores its own bookkeeping goroutines and several runtime/stdlib ones
that are expected to persist: the `testing` package's own goroutines, `signal.Notify`
loops, `os/signal.loop`, the runtime's finalizer and GC goroutines, and a
`created by runtime` background set. Everything else counts as a leak.

Callers extend the filter with `Option`s passed to `VerifyNone` /
`VerifyTestMain`:

- `IgnoreTopFunction(name)` — ignore goroutines whose top-of-stack frame matches
  a fully-qualified function name. The standard escape hatch for a known,
  intentionally-persistent background goroutine (e.g. a connection pool reaper).
- `IgnoreAnyFunction(name)` — ignore if the function appears anywhere in the
  stack, not just the top.
- `IgnoreCurrent()` — snapshot the goroutines alive *now* and exclude them from
  the later check. Useful when a package legitimately starts long-lived
  goroutines before tests run.
- `Cleanup(func(int))` — hook invoked with the exit code, mainly for
  `VerifyTestMain` customization.

Because detection is string-matching against stack text, filters are coupled to
internal function names of your dependencies. That coupling is the architectural
cost of not instrumenting the runtime.

## Production Notes

goleak is a test-only dependency; the caveats are about test reliability, not
serving traffic.

- **`t.Parallel` breaks it.** goleak cannot distinguish a leaked goroutine from a
  sibling parallel subtest that simply hasn't finished. The README calls this out
  explicitly: for packages using `t.Parallel`, use `VerifyTestMain` (which runs
  after everything completes) rather than per-test `VerifyNone`.
- **Third-party background goroutines cause false positives.** Many popular
  libraries keep goroutines alive by design — connection pools, HTTP/2 transports
  (`net/http` keep-alive readers), gRPC, `database/sql` connection reapers,
  loggers with async sinks. You will accumulate a list of `IgnoreTopFunction`
  entries, and those entries reference unexported internals of your deps that can
  change across versions and silently break the ignore.
- **Attribution is manual with `VerifyTestMain`.** When the package-level check
  fails, it does not name the guilty test. The README documents a shell one-liner
  that compiles the test binary and reruns each test in isolation to find the
  culprit — plan for that when a leak appears in a large package.
- **Flakiness under load.** The bounded retry window is short. On a heavily loaded
  CI box, a goroutine that is genuinely shutting down but scheduled late can miss
  the window and produce an intermittent failure. This is uncommon but real.
- **Go version support is narrow.** goleak officially supports only the two most
  recent minor Go releases, matching Go's own policy. It usually works on older
  toolchains, but stack-format assumptions are only guaranteed against current Go.
  The project is v1 and follows SemVer strictly, so the API is stable to pin against.

## When to Use / When Not

**Use when:**
- You maintain a library or service where goroutine lifecycle correctness matters
  (pools, workers, tickers, pub/sub) and want a regression guard.
- You want a near-zero-config check — `defer goleak.VerifyNone(t)` in the tests
  that spawn goroutines, or one `TestMain` per package.
- You're chasing a slow memory growth and suspect leaked goroutines.

**Avoid / be cautious when:**
- Your tests lean heavily on `t.Parallel` and you want per-test granularity —
  you'll fight false positives.
- Your package pulls in dependencies with many intentional background goroutines;
  the maintenance cost of the ignore-list may outweigh the benefit.
- You want production leak monitoring — this is a test-time assertion, not an
  observability tool. Use runtime metrics (`runtime.NumGoroutine`, pprof) for
  live systems.

## Alternatives

- golang/net/http/httptest and stdlib `runtime.NumGoroutine` — hand-rolled
  before/after goroutine counting; use when you want zero dependencies and can
  tolerate manual filtering.
- fortytw2/leaktest — an earlier goroutine-leak checker with the same idea
  (snapshot/diff); use it if you prefer its API, though goleak is more actively
  maintained.
- google/pprof — goroutine profiles for interactive investigation; use when you
  need to find and attribute a leak in a running process rather than assert its
  absence in a test.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-11-02 | Repository created under uber-go. |
| 1.0.0 | 2019 | First tagged release; stable `VerifyNone` / `VerifyTestMain` API. |
| 1.x | 2020–2024 | Incremental options (`IgnoreAnyFunction`, `IgnoreCurrent`, `Cleanup`), Go-version support tracking, stack-parser fixes. |

As of the latest fetch the repo carries roughly 5.2k stars and 166 forks, is MIT
licensed, and was last pushed in mid-2026 — low-volume but actively maintained,
consistent with a mature single-purpose library that needs changes mainly to
track new Go releases.[^1]

## References

[^1]: uber-go/goleak repository and README. https://github.com/uber-go/goleak
[^2]: Package documentation on pkg.go.dev. https://pkg.go.dev/go.uber.org/goleak
[^3]: Go release support policy (goleak tracks the two latest minors). https://go.dev/doc/devel/release#policy

## Tags

go, golang, testing, goroutine, concurrency, leak-detection, test-tooling, uber-go, quality-assurance, developer-tools
