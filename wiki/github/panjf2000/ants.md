# panjf2000/ants

> A fixed-capacity goroutine pool for Go that reuses and recycles workers to cap concurrency and reduce goroutine churn.

[GitHub repo](https://github.com/panjf2000/ants) ·
[Official website](https://ants.andypan.me/) ·
[License: MIT](https://github.com/panjf2000/ants/blob/dev/LICENSE)

## Overview

`ants` is a goroutine pool by Andy Pan (panjf2000, also the author of the
`gnet` networking framework), first published in 2018[^1]. Go makes spawning
goroutines cheap, so the pitch is not "goroutines are expensive to create" but
"unbounded goroutines are expensive to keep": each carries a stack and adds
scheduler and GC pressure, and a program that spawns one per unit of work has no
backpressure. `ants` bounds the live goroutine count to a fixed capacity, reuses
idle workers instead of letting them exit, and purges ones that stay idle too
long. It is one of the most-depended-on concurrency utilities in the Go
ecosystem and is used inside Milvus, TDengine, AdGuard DNS, and go-ethereum
forks, among others[^2].

The defining tension is that a goroutine pool is not always a win. For CPU-bound
or short tasks, a pool can beat naive `go func()` on memory and tail latency; for
tasks that block on I/O, the runtime scheduler already multiplexes blocked
goroutines onto OS threads, and a pool mainly buys you a concurrency ceiling
rather than raw throughput. `ants` is best understood as a concurrency *limiter*
with worker reuse, not as a speedup in the general case. Since Go 1.20,
`errgroup.Group.SetLimit` and channel semaphores cover the "just bound it" case
with no dependency, so the reason to reach for `ants` is worker reuse, panic
handling, runtime capacity tuning, and pool lifecycle control.

## Getting Started

```bash
# v2 (module-aware, current line; requires Go >= 1.19)
go get -u github.com/panjf2000/ants/v2
```

```go
package main

import (
	"fmt"
	"sync"

	"github.com/panjf2000/ants/v2"
)

func main() {
	var wg sync.WaitGroup
	p, _ := ants.NewPool(1000)          // cap live workers at 1000
	defer p.Release()

	for i := 0; i < 10000; i++ {
		wg.Add(1)
		i := i
		_ = p.Submit(func() {
			defer wg.Done()
			fmt.Println(i)
		})
	}
	wg.Wait()
}
```

For a pool bound to a single function (so you submit arguments, not closures),
use `NewPoolWithFunc`; a generics variant `NewPoolWithFuncGeneric` avoids the
`interface{}` boxing[^3].

## Architecture / How It Works

A `Pool` holds a slice of reusable `goWorker`s. Each worker is one long-lived
goroutine running a loop that receives a task over a channel, runs it, then
returns itself to the pool's free list. `Submit` pops a free worker (or blocks /
spawns / rejects, per config) and hands it the task. Because workers are
recycled, N submitted tasks do not create N goroutines — they create at most
`Cap()` of them.

The free-worker container has two implementations. By default it is a **stack**
(LIFO), which favors reusing the most-recently-active worker and lets cold
workers age out. With `WithPreAlloc(true)` it becomes a **ring buffer**
(loopQueue) sized to the capacity up front, trading memory for fewer allocations
in ultra-large, long-task pools. Access to the container is guarded by an
internal **spinlock** rather than a `sync.Mutex`, on the assumption that the
critical section is tiny.

A background **purger** goroutine wakes on an interval and retires workers idle
longer than `ExpiryDuration` (default one second), so a pool that saw a burst
shrinks back down. `WithDisablePurge(true)` keeps workers alive forever — useful
when you want a warm, fixed set and never want them reaped.

Submission has three modes. **Blocking** (the default) makes `Submit` wait for a
free worker when the pool is full; `WithMaxBlockingTasks` caps how many callers
may wait before further submits get `ErrPoolOverload`. **Nonblocking**
(`WithNonblocking(true)`) never waits — it returns `ErrPoolOverload` the instant
the pool is saturated. Panics inside tasks are recovered so one bad task cannot
crash the process; `WithPanicHandler` lets you observe them instead of silently
logging. To reduce lock contention on many-core machines there is also a
`MultiPool` that shards submissions across several underlying pools.

## Production Notes

- **Blocking Submit can deadlock.** The default mode blocks when the pool is
  full. If a task submits another task to the *same* pool and waits on its
  result, a full pool can deadlock — all workers are parked waiting on work that
  can never be scheduled. Never fan out into the same pool you are blocked on;
  use a separate pool or a nonblocking mode.
- **`Tune` is a no-op under `WithPreAlloc`.** Runtime capacity tuning works on
  the default stack pool; a pre-allocated ring-buffer pool has a fixed size and
  ignores `Tune`. Decide between elastic and pre-allocated up front.
- **The pool bounds goroutines, not queued work.** In blocking mode without
  `WithMaxBlockingTasks`, an unbounded number of *callers* can be parked waiting
  to submit. The live-goroutine ceiling holds, but memory from blocked callers
  and their captured closures does not — set a blocking cap or use nonblocking
  with explicit backpressure.
- **Silent panics.** By default a panicking task is recovered and logged through
  the pool's logger; without `WithPanicHandler` you can lose failures. Always set
  a handler in production so errors reach your telemetry.
- **Reuse means shared state lingers.** A recycled worker's goroutine persists
  across tasks, so goroutine-local patterns (e.g. anything keyed off the stack or
  `pprof` labels set inside a task) may bleed between unrelated tasks. Reset such
  state at the top of each task.
- **`Release` then `Reboot`.** A released pool rejects submits; call `Reboot` to
  make it usable again. `ReleaseTimeout` waits for in-flight tasks to drain
  rather than returning immediately.
- **v1 vs v2 import paths differ.** v1 is `github.com/panjf2000/ants`; v2 is
  `.../ants/v2`. They are distinct modules — mixing both pulls two copies. New
  code should use v2.

## When to Use / When Not

**Use when:**
- You need a hard ceiling on concurrent goroutines for CPU- or memory-heavy work.
- You submit a very large number of short tasks and want to avoid per-task
  goroutine/stack churn and GC pressure.
- You want built-in panic recovery, runtime capacity tuning, or explicit
  overload rejection as a backpressure signal.

**Avoid when:**
- Your tasks are mostly I/O-bound: the runtime already schedules blocked
  goroutines efficiently, and a pool adds a ceiling more than throughput.
- You just need "bound concurrency to N" with no dependency — `errgroup` with
  `SetLimit` or a buffered-channel semaphore is simpler and stdlib-adjacent.
- You need ordered results or per-task cancellation/timeouts — `ants` does not
  guarantee execution order[^4] and leaves context handling to the task body.

## Alternatives

- alitto/pond — worker pool with contexts, generics, and metrics; use when you
  want richer task ergonomics and observability out of the box.
- gammazero/workerpool — smaller, simpler dynamic pool; use when you want minimal
  surface area over correctness knobs.
- sourcegraph/conc — structured concurrency (`pool`, `stream`, safe `WaitGroup`);
  use when you want panic-safe fan-out with typed results rather than a
  long-lived pool.
- golang.org/x/sync — `errgroup.SetLimit` / `semaphore.Weighted`; use when you
  only need a concurrency limit and error propagation with no third-party dep.
- Jeffail/tunny — older synchronous goroutine pool; use when you want a
  blocking process-a-payload API and don't need `ants`'s tuning.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0 | 2018 | Initial release: fixed-capacity pool, worker reuse, purging[^1]. |
| v2.0 | 2019 | Go-modules-aware rewrite under the `/v2` import path. |
| v2.x | 2020–2024 | Functional options, nonblocking mode, `MultiPool`, generics func pool, min Go bumped to 1.19[^3]. |

Repository stats at time of writing: ~14.5k stars, ~1.4k forks, MIT-licensed,
last pushed 2026-07-04 — actively maintained with a near-empty issue tracker,
consistent with a small, stable, well-scoped library rather than an expanding
platform[^5].

## References

[^1]: `ants` repository and author's write-up on the goroutine-pool design.
  https://github.com/panjf2000/ants and
  https://taohuawu.club/high-performance-implementation-of-goroutine-pool
[^2]: "Use cases" section of the README listing adopting organizations and
  open-source projects. https://github.com/panjf2000/ants#-use-cases
[^3]: Package API reference (`Options`, `NewPoolWithFunc`,
  `NewPoolWithFuncGeneric`, minimum Go version).
  https://pkg.go.dev/github.com/panjf2000/ants/v2
[^4]: README, "About sequence": submitted tasks are not guaranteed to run in
  order. https://github.com/panjf2000/ants#%EF%B8%8F-about-sequence
[^5]: GitHub API `repos/panjf2000/ants`, fetched 2026-07-15.

## Tags

go, goroutine-pool, worker-pool, concurrency, goroutine, thread-pool, backpressure, performance, library, panjf2000
