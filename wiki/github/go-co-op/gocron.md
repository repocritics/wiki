# go-co-op/gocron

> In-process job scheduler for Go — durations, cron expressions, and calendar intervals, with optional leader election and distributed locking.

[GitHub repo](https://github.com/go-co-op/gocron) ·
[License: MIT](https://github.com/go-co-op/gocron/blob/v2/LICENSE)

## Overview

gocron runs Go functions on a schedule inside your own process. It is not a
daemon, a queue, or a cron replacement at the OS level: you construct a
`Scheduler`, register jobs (a Go func plus its arguments), and the scheduler
fires each job on its own goroutine at the interval you describe. Intervals can
be fixed durations, random durations, standard five/six-field crontab strings,
or calendar rules (daily / weekly / monthly at specific times)[^1].

The project began in 2020 as a fork of the now-inactive `jasonlvhit/gocron`[^2]
and is maintained by the go-co-op organization. Its defining event is the v2
rewrite: v1 exposed a fluent builder (`s.Every(1).Day().At("10:30").Do(fn)`),
while v2 replaced it with a functional-options API (`NewScheduler(...)`,
`NewJob(DurationJob(...), NewTask(fn, args...))`). The two are not
source-compatible, and v1 and v2 live under different import paths
(`.../gocron` vs `.../gocron/v2`). Most current documentation, examples, and
maintenance target v2; v1 still exists but should be treated as legacy.

The central tradeoff is scope. gocron deliberately stays in-process and
in-memory: there is no built-in persistence, no retry queue, and no dashboard in
the core library. That keeps it small and dependency-light, but it means
"survives a restart," "runs exactly once across N replicas," and "retries on
failure" are things you assemble yourself from the distributed primitives it
provides, or reach for a queue-based system instead.

## Getting Started

```bash
go get github.com/go-co-op/gocron/v2
```

```go
package main

import (
	"fmt"
	"time"

	"github.com/go-co-op/gocron/v2"
)

func main() {
	s, err := gocron.NewScheduler()
	if err != nil {
		panic(err)
	}

	// Run every 10 seconds; the task is a func plus its bound arguments.
	j, err := s.NewJob(
		gocron.DurationJob(10*time.Second),
		gocron.NewTask(func(name string) {
			fmt.Println("hello", name)
		}, "world"),
	)
	if err != nil {
		panic(err)
	}
	fmt.Println("job id:", j.ID())

	s.Start()                 // non-blocking; jobs run on their own goroutines
	time.Sleep(time.Minute)   // keep main alive
	_ = s.Shutdown()          // stop and wait for in-flight jobs
}
```

A cron-driven job is `gocron.CronJob("0 * * * *", false)` (the bool enables
seconds as a leading sixth field). `s.Start()` returns immediately, so a
long-lived process must block on its own — a bare scheduler in `main` with no
blocking call will exit before any job fires.

## Architecture / How It Works

The README names three roles[^1]. The **Scheduler** owns the job set and decides
when each job is next due. Each **Job** wraps a **Task** (the func + arguments)
and knows how to compute its next run time from its schedule definition. The
**Executor** actually invokes tasks and enforces concurrency rules.

Timing is driven by per-job Go timers rather than a single tick loop, so a job's
next fire is computed from its own schedule. By default the next run is measured
from the *scheduled* start time, which yields stable wall-clock intervals even
when a run takes a while; `WithIntervalFromCompletion` instead measures from when
the previous run *finished*, giving a fixed rest gap — the right choice for
rate-limited APIs or heavy jobs where overlap must be avoided[^1].

Concurrency is controlled at two levels. **Singleton mode**
(`WithSingletonMode`) limits a single job to one execution at a time, either
rescheduling (skip the overlap) or queueing (wait). **Limit mode**
(`WithLimitConcurrentJobs`) caps concurrent executions across the whole
scheduler. Both can be enabled together.

Distributed operation is opt-in through two interfaces. An **Elector**
(`WithDistributedElector`) elects one instance as primary so only the leader
runs jobs; the others stand by. A **Locker** (`WithDistributedLocker`) instead
lets every instance schedule but takes a lock per run so a given execution
happens once cluster-wide. These are interfaces, not implementations: concrete
Redis/Postgres/etc. electors and lockers live in separate `go-co-op/*-elector`
and `*-lock` repositories[^3]. Time is injected through a `Clock`
(`WithClock`), backed by `jonboulle/clockwork`, which is what makes schedules
deterministically testable with a `FakeClock`[^1].

## Production Notes

- **No persistence.** Jobs live in memory. Restart the process and every
  schedule resets to its next-from-now fire; there is no "catch up on missed
  runs" and no run history. If a job must survive restarts or fire exactly on a
  missed slot, persist that state yourself or use a queue-backed system.
- **Distributed correctness is only as good as the plugin.** The Elector and
  Locker are interfaces; guarantees depend on the concrete backend (its lock
  TTL, clock assumptions, and failure behavior). The docs explicitly flag design
  limitations for the Locker — a lock acquired but not released before TTL, or
  clock skew across nodes, can still allow a double-run or a skipped run[^1].
  Read the Locker notes before relying on "runs once cluster-wide."
- **Errors are yours to handle.** A task that panics or returns an error does
  not retry by default. Use `WithEventListeners` / event listeners
  (`AfterJobRunsWithError`, etc.) to observe failures and implement retry,
  alerting, or backoff. There is no built-in dead-letter or retry policy.
- **Observability needs wiring.** The `Monitor` / `MonitorStatus` /
  `SchedulerMonitor` interfaces expose job and lifecycle events (execution time,
  scheduling delay, started/completed/failed), but as of this writing the README
  states there are no open-source implementations shipped — you implement the
  interface (e.g. a Prometheus adapter) yourself[^1].
- **v1 → v2 is a rewrite, not an upgrade.** Migrating means rewriting scheduling
  code against a different API and changing the import path to `/v2`. Budget for
  it; there is no drop-in shim.
- **Graceful shutdown matters.** `Shutdown()` (or `ShutdownWithContext`) stops
  scheduling and waits for in-flight jobs; skipping it can drop or truncate
  running tasks on exit.

## When to Use / When Not

**Use when:**
- You need scheduled work inside a single Go service (cleanup, polling,
  digests, cache warming) without standing up external infrastructure.
- You want cron plus richer calendar/duration/random intervals in one API.
- You have a small cluster and leader election or per-run locking is enough to
  avoid duplicate runs, and you can supply a backend for it.

**Avoid when:**
- You need durable jobs, retries, backoff, and a queue that survives crashes —
  use a task-queue system instead.
- You want a UI, run history, or dead-letter handling out of the box (the
  separate `gocron-ui` dashboard exists but is not the core library).
- Exactly-once execution across nodes is a hard requirement and you cannot
  accept the caveats of an external locker.
- You only need OS-level periodic execution — plain `cron`/systemd timers are
  simpler than embedding a scheduler.

## Alternatives

- robfig/cron — the long-standing minimal Go cron scheduler; use it when you
  want just crontab parsing and in-process firing with no distributed features.
- hibiken/asynq — Redis-backed distributed task queue with scheduling, retries,
  and a web UI; use it when jobs must be durable and survive restarts.
- reugn/go-quartz — Quartz-inspired scheduler with a job/trigger model; use it
  when you prefer that abstraction or need in-memory job stores.
- procyon-projects/chrono — Spring-style scheduling API; use it if you want a
  familiar fixed-rate/fixed-delay/cron surface.
- OS cron / systemd timers — use these when the work is a standalone command and
  you don't need it embedded in a Go process at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| fork | 2020-03 | go-co-op forks the inactive jasonlvhit/gocron[^2]. |
| v1.x | 2020–2024 | Fluent builder API (`s.Every(1).Day().Do(fn)`); maintenance-mode legacy. |
| v2.0.0 | 2024 | Ground-up rewrite: functional-options API, `NewScheduler`/`NewJob`, distributed Elector/Locker, `/v2` import path[^1]. |
| v2.x | 2024–2026 | Interval-from-completion, scheduler monitor events, ongoing releases; `v2` is the default branch. |

## References

[^1]: gocron README and package documentation. https://pkg.go.dev/github.com/go-co-op/gocron/v2
[^2]: Original project (inactive), from which go-co-op/gocron was forked. https://github.com/jasonlvhit/gocron
[^3]: go-co-op organization — elector and locker implementation repositories. https://github.com/go-co-op

## Tags

go, golang, job-scheduler, cron, scheduling, distributed-systems, leader-election, in-process, task-scheduling, library
