# robfig/cron

> An in-process cron-spec job scheduler for Go — the de facto standard for "run this function on a schedule" inside a single process.

[GitHub repo](https://github.com/robfig/cron) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/github.com/robfig/cron/v3) ·
[License: MIT](https://github.com/robfig/cron/blob/master/LICENSE)

## Overview

`robfig/cron` parses cron specifications and runs Go functions when they are due, all inside the calling process. It has been the default answer to "how do I schedule a job in Go" since 2012[^1], and at ~14k stars is the most-imported cron library in the ecosystem. The API surface is small: construct a `Cron`, `AddFunc`/`AddJob` with a spec string, `Start()`, and jobs fire in their own goroutines.

The defining characteristic is that it is *only* a scheduler, not a job system. There is no persistence, no distributed coordination, no missed-run catch-up, and no built-in retry. If the process restarts, the schedule is rebuilt from code but any run that "should have happened" while it was down is silently skipped. This is the right tradeoff for a sidecar timer inside one long-lived binary, and the wrong one for anything that needs durability or horizontal scale.

The other thing to know up front is maintenance cadence. The library is stable and widely depended-on, but slow-moving: the last release, v3.0.0, shipped in June 2019[^2], and the default branch has seen only sporadic commits since. As of 2026 there are ~175 open issues. Treat it as "done and dependable" rather than "actively developed" — most real bugs were addressed in the v3 merge, and the API has not needed to change.

## Getting Started

```bash
go get github.com/robfig/cron/v3@v3.0.0
```

The `/v3` import path is mandatory — it is a Go-modules major version and requires Go 1.11+[^2].

```go
package main

import (
	"fmt"
	"github.com/robfig/cron/v3"
)

func main() {
	c := cron.New()                          // standard 5-field spec: min hour dom mon dow
	c.AddFunc("*/5 * * * *", func() {        // every 5 minutes
		fmt.Println("tick")
	})
	c.AddFunc("@hourly", func() { fmt.Println("top of the hour") })
	c.Start()
	select {}                                 // block; c.Stop() halts scheduling (not running jobs)
}
```

To use the six-field spec (leading seconds column) that v1 accepted by default, opt in explicitly:

```go
c := cron.New(cron.WithSeconds())            // "30 */5 * * * *" — sec min hour dom mon dow
```

## Architecture / How It Works

A `Cron` holds a slice of `Entry` values, each pairing a `Schedule` (the parsed spec) with a `Job` (anything implementing `Run()`). Scheduling is a single goroutine running a loop: it sorts entries by next-fire time, sleeps on a timer until the soonest one, then launches every due job with `go job.Run()` and recomputes the next times. Mutations (`AddFunc`, `Remove`, `Entries`) are marshalled to that goroutine over channels, so the design is concurrency-safe without exposing locks.

Because every fire is `go job.Run()`, **there is no serialization between runs of the same entry**. A job that takes longer than its interval will overlap itself. Controlling that is the job of the `Chain`/`JobWrapper` mechanism added in v3: `cron.WithChain(cron.SkipIfStillRunning(logger))` drops a run if the previous one is in flight, `DelayIfStillRunning` queues it, and `cron.Recover(logger)` restores panic recovery[^2]. These wrap the `Job` at add-time; nothing is global.

Spec parsing is pluggable via the `Parser` type. The default (`cron.New()`) is standard 5-field cron plus the `@hourly`/`@daily`/`@every 1h30m` descriptors. `WithParser` lets you assemble a custom field set from bit flags (`Second | Minute | Hour | Dom | Month | Dow | Descriptor`), which is how `WithSeconds` and Quartz-style compatibility are built. The Quartz "year" field is not supported.

Timezones are per-`Cron` via `cron.WithLocation(loc)`, or per-entry with a `CRON_TZ=America/New_York` prefix in the spec string (the sanctioned form); the legacy `TZ=` prefix is still honored[^2]. DST handling was one of the main fixes in v3: clock-forward transitions no longer skip a non-existent local midnight, and clock-back no longer double-fires.

## Production Notes

**No durability, no catch-up.** The schedule lives entirely in memory. A deploy, crash, or restart means every run that was due during downtime is lost with no record — there is no missed-execution replay. If a nightly billing job *must* run even if the box was down at 2 a.m., this library alone will not give you that; you need external durable state or a real job queue.

**Overlap is the classic footgun.** New users assume runs are serialized. They are not. A job whose runtime exceeds its period will pile up goroutines. Always wrap long jobs with `SkipIfStillRunning` or `DelayIfStillRunning`, and remember these wrappers are per-entry — adding one to the `Cron` chain covers all entries, adding it at `AddJob` covers one.

**Panic recovery is OFF by default in v3.** In v1 a panicking job was recovered silently; v3 removed that because it was surprising and un-idiomatic[^2]. A single unhandled panic in a job goroutine crashes the whole process. If you want the old behavior, add `cron.Recover(logger)` to the chain explicitly.

**The v1→v3 seconds break.** v1's default spec had a leading seconds field; v3's default does not. Upgrading without `WithSeconds()` silently reinterprets every existing spec — `"0 30 * * * *"` (v1: :30 seconds) becomes an invalid or shifted 5-field spec. This is the single most common upgrade bug. Audit every spec string when moving to v3.

**`Stop()` does not wait for running jobs.** It stops the scheduler goroutine and returns a `context.Context` that is done once in-flight jobs finish (v3.1+ semantics via `c.Stop().Done()` in later commits); the base v3.0.0 `Stop()` returns immediately without draining. For graceful shutdown you must track in-flight work yourself or block on that context if your version exposes it.

**Multi-instance = multi-fire.** Running the same binary on three pods means the job fires three times. There is no leader election. Deployments that scale horizontally need an external lock (advisory DB lock, Redis lease) around the job body, or a different tool entirely.

## When to Use / When Not

**Use when:**
- You need periodic in-process timers inside a single long-running Go service (cache refresh, metrics flush, cleanup sweeps).
- The work is idempotent or tolerant of an occasional missed or duplicated run.
- You want a tiny, dependency-light, well-understood library with a stable API.

**Avoid when:**
- Jobs must survive restarts or be guaranteed to run exactly once — you need a durable queue.
- You run multiple replicas and can't tolerate N-fold execution without adding your own locking.
- You need retries, backoff, dead-lettering, observability, or a job history — that is a task-queue's job, not a cron ticker's.

## Alternatives

- go-co-op/gocron — use when you want a fluent, chainable scheduling API (`Every(5).Minutes()`), optional distributed locking, and more active maintenance.
- hibiken/asynq — use when you need durable, distributed, Redis-backed task processing with retries and a dashboard; cron scheduling is one feature of a real queue.
- reugn/go-quartz — use when you want a Quartz-style scheduler with in-memory job storage and trigger abstractions closer to the Java ecosystem.
- madflojo/tasks — use when you want a minimal scheduler keyed on `time.Duration` intervals rather than cron strings.
- Native `time.Ticker` / `time.AfterFunc` — use when your schedule is a fixed interval and you don't need cron-spec parsing at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.x | 2012–2017 | Original API; default spec included a leading seconds field[^1]. |
| v2 (branch) | ~2017 | Added job removal; never tagged as a module major version. |
| v3.0.0 | 2019-06 | Go modules (`/v3` path), standard 5-field default, `Chain`/`JobWrapper`, `logr`-style logging, DST fixes, panic recovery off by default[^2]. |
| (master) | 2024-07 | Sporadic post-v3 fixes on `master`; no new tagged release. |

## References

[^1]: `robfig/cron` repository, created 2012-07-06; `README` and package history. https://github.com/robfig/cron
[^2]: `robfig/cron` README, "Upgrading to v3 (June 2019)" — breaking changes, Chain/JobWrapper, seconds-field default, panic-recovery and timezone notes. https://github.com/robfig/cron/blob/master/README.md

## Tags

go, cron, scheduler, job-scheduling, cron-expression, in-process, background-jobs, task-scheduler, concurrency, library
