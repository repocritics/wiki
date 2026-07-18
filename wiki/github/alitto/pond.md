# alitto/pond

> A dependency-free Go worker pool that caps concurrency and scales its goroutines to zero when idle.

[GitHub repo](https://github.com/alitto/pond) ·
[API reference](https://pkg.go.dev/github.com/alitto/pond/v2) ·
[License: MIT](https://github.com/alitto/pond/blob/main/LICENSE)

## Overview

pond is a small Go library implementing the worker-pool (thread-pool) pattern: you
submit an arbitrary number of tasks, and pond runs them across a bounded set of
goroutines so you never spawn ten thousand of them at once[^1]. It exists to solve
resource exhaustion and rate-limiting problems — bounding concurrent HTTP calls,
DB connections, or outbound requests to a throttled API — without you hand-rolling
a semaphore and a `sync.WaitGroup` every time. It has zero external dependencies.

The defining trait is that workers are created lazily and removed once idle, so an
empty pool costs nothing (scale-to-zero). The README claims pond can outperform
raw unbounded goroutines under heavy load, which is plausible because a bounded set
of long-lived workers avoids the scheduler churn and stack-allocation cost of
spawning a goroutine per task[^1].

The important context for anyone evaluating pond in 2026 is the **v1 → v2 split**.
v2 is a near-total API rewrite built around Go generics: typed result pools,
awaitable tasks (`task.Wait()`), panic-to-error recovery, subpools, dynamic
resizing, and bounded queues all arrived in v2[^2]. The module path changed to
`github.com/alitto/pond/v2`, and most v1 code does not compile against it
unchanged. Treat the two versions as different libraries that share a name.

## Getting Started

```bash
go get -u github.com/alitto/pond/v2
```

```go
package main

import (
	"fmt"

	"github.com/alitto/pond/v2"
)

func main() {
	// Pool that runs at most 100 tasks concurrently.
	pool := pond.NewPool(100)

	for i := 0; i < 1000; i++ {
		i := i
		pool.Submit(func() {
			fmt.Printf("running task #%d\n", i)
		})
	}

	// Block until every submitted task has finished.
	pool.StopAndWait()
}
```

Tasks that return a value or error use the typed variants:

```go
pool := pond.NewResultPool[string](10)

task := pool.SubmitErr(func() (string, error) {
	return "Hello, World!", nil
})

result, err := task.Wait() // result = "Hello, World!", err = nil
```

## Architecture / How It Works

A pool holds a maximum-concurrency limit and an internal task queue. Submitting a
task enqueues it; the pool spawns a worker goroutine on demand, up to the limit,
and each worker pulls tasks from the queue until the queue drains, at which point
idle workers exit. This is what makes an idle pool free — there is no fixed pool of
parked goroutines the way `panjf2000/ants` keeps a reusable set.

v2's public surface is built on generics. `NewPool(n)` returns an untyped pool;
`NewResultPool[T](n)` returns a pool whose tasks yield a `T`. Both `Submit` /
`SubmitErr` return a task handle you can `Wait()` on — pond synthesizes a future
per task, so awaiting is opt-in rather than pool-wide. This is a genuine
convenience over stdlib `errgroup`, which only gives you a single joined result.

Key internals worth understanding:

- **Task queue.** Unbounded by default — tasks accumulate until the pool stops or
  the process runs out of memory. `WithQueueSize(n)` makes it bounded;
  `WithQueueSize(0)` disables queuing entirely (tasks run immediately or are
  rejected); `pond.Unbounded` restores the default.
- **Panic recovery is on by default.** Each task runs inside a `recover()`; a panic
  becomes the error returned by `task.Wait()` rather than crashing the process.
  `WithoutPanicRecovery()` restores standard Go crash semantics.
- **Subpools** (`pool.NewSubpool(k)`) borrow a fraction of the parent's worker
  budget rather than owning independent workers, so a subpool competes with its
  parent and siblings for the same concurrency limit.
- **Context propagation.** A pool takes an optional context (`WithContext`);
  cancelling it stops workers and drains the queue, resolving queued tasks with a
  cancellation error instead of running them.
- **Metrics** are plain atomic counters (`RunningWorkers`, `WaitingTasks`,
  `SubmittedTasks`, `CompletedTasks`, `DroppedTasks`, and so on) read directly off
  the pool — cheap to poll, suitable for a Prometheus collector.

## Production Notes

**The default unbounded queue is a memory footgun.** If producers outrun the
worker limit, tasks pile up in the queue with no backpressure until the process
OOMs. Any pool fed by an unbounded or bursty source should set `WithQueueSize` plus
either `TrySubmit` or `WithNonBlocking(true)` so full-queue submissions fail fast
with `ErrQueueFull` instead of silently growing. Non-blocking mode is a no-op on an
unbounded queue — submission always "succeeds" there.

**Group first-error semantics are subtle.** `group.Wait()` returns on the first
error, and queued (not-yet-started) tasks are aborted — but tasks already running
are *not* interrupted. `Wait()` returns while they keep executing in the
background. To actually stop in-flight work you must thread `group.Context()` into
each task and check it during long operations. Missing this leads to "cancelled"
groups whose goroutines are still doing work and holding resources.

**Panic recovery changes debugging.** Because panics are converted to errors by
default, a task that panics produces no stack-trace crash — the failure only
surfaces if you `Wait()` on the task and inspect the error. Fire-and-forget
`Submit` without a `Wait` will swallow the panic entirely. Consider
`WithoutPanicRecovery()` in environments where you rely on crash-on-panic and
process supervisors.

**It is a concurrency limiter, not a job system.** pond has no retries, no
backoff, no scheduling, no persistence, and no distribution. Tasks live in memory;
a process restart loses the queue. If you need durable or scheduled jobs, pond is
the wrong layer.

**Resizing is cooperative on shrink.** `Resize(n)` grows immediately by allowing
new workers, but shrinking does not preempt running workers — they finish their
current task and no new workers start until the running count falls below the new
limit. Expect a lag, not an instant reduction.

**v1 → v2 is a real migration, not a bump.** The import path gains `/v2`,
`pond.New(max, capacity)` becomes `pond.NewPool(max)` (queue is unbounded by
default so the second arg is gone), `pond.Context` becomes `pond.WithContext`, and
`MinWorkers` is removed[^2]. Budget time; it is not a find-and-replace.

## When to Use / When Not

**Use when:**
- You need to cap concurrent goroutines for I/O-bound fan-out (HTTP, DB, rate-limited APIs).
- You want typed results and per-task `Wait()` without wiring channels yourself.
- You want zero dependencies and a small, idiomatic API.
- You want per-task panic isolation as errors rather than process crashes.

**Avoid when:**
- You need durable, retryable, or scheduled jobs — reach for a real queue.
- Your workload is a simple bounded fan-out with one joined error — stdlib `errgroup` with `SetLimit` is enough.
- You need maximum raw throughput with pooled goroutine reuse — `ants` is more specialized there.
- You are on v1 and cannot afford the v2 migration; pin v1 explicitly and know it is in maintenance.

## Alternatives

- panjf2000/ants — high-throughput pool built on goroutine reuse via sync.Pool; use it when peak tasks/sec and goroutine recycling matter more than a fluent typed API.
- sourcegraph/conc — structured-concurrency helpers (`conc.WaitGroup`, `pool.Pool`); use it when you want ergonomic scoped concurrency rather than a long-lived pool object.
- golang.org/x/sync/errgroup — stdlib-adjacent; use it for simple bounded fan-out (`SetLimit`) with a single joined error and no extra dependency.
- gammazero/workerpool — minimal pre-generics worker pool; use it when you want a tiny untyped API and don't need results or subpools.
- alitto/pond (v1) — the pre-generics line; use it only for legacy code you cannot migrate.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-03 | First release; v1 worker pool with fixed queue capacity[^1]. |
| v1.x | 2020–2024 | Iterative v1 line: metrics, context support, resizing groundwork. |
| v2.0 | 2024 | Generics rewrite: result pools, awaitable tasks, panic recovery, subpools, bounded queues, dynamic resize; module path `/v2`[^2]. |

## References

[^1]: pond README — motivation, features, and usage. https://github.com/alitto/pond
[^2]: pond README — "Migrating from pond v1 to v2" and v2 feature list. https://github.com/alitto/pond#migrating-from-pond-v1-to-v2

## Tags

go, golang, concurrency, worker-pool, goroutines, generics, rate-limiting, thread-pool, task-queue, zero-dependencies
