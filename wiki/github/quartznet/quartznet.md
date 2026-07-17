# quartznet/quartznet

> A full-featured job scheduler for .NET — cron/interval triggers, durable jobs, and database-backed clustering, ported from Java's Quartz Scheduler.

[GitHub repo](https://github.com/quartznet/quartznet) ·
[Official website](https://www.quartz-scheduler.net/) ·
[License: Apache-2.0](https://github.com/quartznet/quartznet/blob/main/license.txt)

## Overview

Quartz.NET is a port of the Java Quartz Scheduler[^1] to .NET, and has been the default answer to "how do I run a scheduled job in .NET" since the mid-2000s. It schedules arbitrary units of work (jobs) against triggers — either simple interval repeats or Unix-style cron expressions — and can persist that schedule to a database so it survives process restarts and coordinates across a cluster of nodes. It is distributed on NuGet as the `Quartz` package plus a family of extension packages for hosting and dependency injection.

The library targets .NET (Core) via netstandard 2.0 and .NET Framework 4.6.2 and later[^2], so it runs on both the modern and legacy runtimes. The 3.x line (a substantial rewrite) is async-first: jobs return `Task`, and the scheduler is built around `async`/`await` rather than the thread-per-job model of the 2.x era.

The defining tension is scope. Quartz.NET is a scheduler, not a background-job queue. It excels at "run this at 02:00 every weekday" and at cluster-safe, database-durable scheduling with misfire recovery — but it has no built-in dashboard, no fire-and-forget enqueue API, and no first-class retry-with-backoff of individual job invocations. Teams that actually want a job queue with a UI frequently reach for Hangfire instead and find Quartz's trigger/job/jobstore model heavier than they needed.

## Getting Started

```bash
dotnet add package Quartz
dotnet add package Quartz.Extensions.Hosting
```

```csharp
// Program.cs — ASP.NET Core / Generic Host integration
using Quartz;

builder.Services.AddQuartz(q =>
{
    var jobKey = new JobKey("ReportJob");
    q.AddJob<ReportJob>(opts => opts.WithIdentity(jobKey));
    q.AddTrigger(opts => opts
        .ForJob(jobKey)
        .WithIdentity("ReportJob-trigger")
        .WithCronSchedule("0 0 2 ? * MON-FRI"));   // 02:00, weekdays
});
builder.Services.AddQuartzHostedService(opts => opts.WaitForJobsToComplete = true);

public class ReportJob : IJob
{
    public async Task Execute(IJobExecutionContext context)
    {
        await Task.CompletedTask; // do work here
    }
}
```

The cron field order is Quartz's 6/7-field form (seconds first, plus optional year) — it is **not** standard Unix 5-field cron, a routine source of confusion.

## Architecture / How It Works

Three abstractions carry the model:

1. **Jobs** — classes implementing `IJob`. A fresh instance is constructed for each execution (via the DI-aware job factory in modern setups), so jobs are stateless by default. Data is passed through `JobDataMap`. `[DisallowConcurrentExecution]` serializes runs of the same job key; `[PersistJobDataAfterExecution]` writes the job's data map back after each run.
2. **Triggers** — `SimpleTrigger` (interval + repeat count) and `CronTrigger` (cron expression + time zone). A trigger references exactly one job; a job may have many triggers.
3. **JobStore** — where jobs and triggers live. `RAMJobStore` is the in-memory default: fast, zero-config, and lost on restart. `AdoJobStore` (`JobStoreTX`) persists everything to a relational database.

The scheduler pulls due triggers from the jobstore, hands them to a thread pool, and updates trigger state. With `AdoJobStore`, this loop runs through SQL against a fixed set of tables (default prefix `QRTZ_`), and cluster coordination is achieved by pessimistic row locking on a locks table — every node polls the same database and uses `SELECT ... FOR UPDATE`-style locking so exactly one node fires each trigger[^3].

Supported backends for `AdoJobStore` include SQL Server, PostgreSQL, MySQL/MariaDB, Oracle, SQLite, and Firebird, each via a delegate class that papers over SQL dialect differences. Persisted `JobDataMap` values are serialized; the JSON serializer (`Quartz.Serialization.Json`) is the recommended option, as binary serialization is deprecated on modern .NET.

Everything is wired through `Quartz.Extensions.DependencyInjection` and `Quartz.Extensions.Hosting` for `Microsoft.Extensions.Hosting` apps, so the scheduler starts and stops with the host lifecycle.

## Production Notes

**Clustering needs a shared database and synchronized clocks.** Clustering only works with `AdoJobStore`, never `RAMJobStore`. All nodes must point at the same database, set `quartz.jobStore.clustered = true`, and each carry a distinct `InstanceId` (`AUTO` generates one). Node clocks must be closely synchronized — the failover/misfire logic compares timestamps across nodes, and skew causes premature failover or missed fires. There is no leader election beyond database row locks.

**Misfires are a policy you must choose.** When a trigger's fire time passes without the scheduler running it (downtime, thread starvation, a long-running previous job), it "misfires." The misfire threshold (default 60s) and per-trigger misfire instructions (fire-now, do-nothing, reschedule) determine what happens. Leaving these at defaults on a job that matters is a common way to silently skip runs after a deploy or an outage.

**Thread pool sizing gates concurrency.** The default `maxConcurrency` is small (historically 10). A burst of simultaneously-due triggers queues behind the pool; combined with `WaitForJobsToComplete = true` at shutdown, a slow job can delay host termination. Size the pool for your peak simultaneous fire count.

**Cron is Quartz-flavored.** Six or seven fields, seconds-first, with `?`, `L`, `W`, and `#` extensions. Day-of-week is 1–7 with Sunday = 1. Time zones are per-trigger; a trigger with no explicit zone uses the server's local zone, which makes DST transitions and server-migration behavior easy to get wrong.

**Schema migrations are manual.** The `QRTZ_` tables are created from SQL scripts shipped in the repo, not by an automatic migration. Major upgrades occasionally change the schema, so upgrading a clustered, persisted deployment means applying DDL as a deliberate step and, ideally, draining nodes first.

**No dashboard.** Observability is your responsibility — hook the scheduler/job/trigger listener interfaces (`IJobListener`, `ITriggerListener`, `ISchedulerListener`) to emit logs and metrics. There is no built-in UI comparable to Hangfire's.

## When to Use / When Not

**Use when:**
- You need cron-style or calendar-based scheduling in a .NET service.
- Schedules must survive restarts and coordinate across multiple instances without double-firing.
- You want misfire recovery and durable triggers backed by a database you already run.
- You need per-job concurrency control and rich trigger semantics (calendars, exclusions, time zones).

**Avoid when:**
- You want fire-and-forget background jobs with retries and a monitoring UI — Hangfire fits that shape better.
- Your need is a single periodic loop — a `BackgroundService` with `PeriodicTimer` is far less machinery.
- You have no database and no clustering requirement; `RAMJobStore` works but so would something lighter.
- You want workflow orchestration (multi-step, stateful, long-running) — that is Temporal/Elsa territory, not a scheduler.

## Alternatives

- HangfireIO/Hangfire — use instead when you want persistent fire-and-forget jobs, automatic retries, and a built-in dashboard rather than cron-centric scheduling.
- jamesmh/coravel — use for small/medium apps wanting a lightweight fluent scheduler with zero external dependencies.
- fluentscheduler/FluentScheduler — use when you need simple in-memory interval scheduling and can live without persistence or clustering.
- dotnet/runtime (`BackgroundService` + `PeriodicTimer`) — use when a single in-process periodic loop is all you need, no durability or coordination.
- temporalio/temporal — use when the real problem is durable, multi-step workflow orchestration, not point-in-time job scheduling.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | ~2009 | Initial .NET port of Java Quartz; SimpleTrigger/CronTrigger, RAMJobStore, AdoJobStore. |
| 2.0 | ~2011 | API modernization, job/trigger builder fluent API, improved clustering. |
| 3.0 | ~2017 | Async/await rewrite; netstandard 2.0 + .NET Core support; DI/hosting extensions[^2]. |
| 3.x | ongoing | JSON serialization, expanded database delegates, continued .NET target updates[^4]. |

Exact release dates for older majors are approximate; consult the GitHub releases page for authoritative dates[^4].

## References

[^1]: Quartz Scheduler (Java), the upstream project Quartz.NET is ported from. https://www.quartz-scheduler.org/
[^2]: Quartz.NET README — compatibility (netstandard 2.0, .NET Framework 4.6.2+). https://github.com/quartznet/quartznet
[^3]: Quartz.NET documentation — clustering and AdoJobStore. https://www.quartz-scheduler.net/documentation/
[^4]: Quartz.NET releases. https://github.com/quartznet/quartznet/releases

## Tags

csharp, dotnet, job-scheduler, cron, scheduling, background-jobs, clustering, quartz, nuget, task-scheduler
