# HangfireIO/Hangfire

> Background job processing for .NET, backed by your existing database — no separate service, queue broker, or Windows Service required.

[GitHub repo](https://github.com/HangfireIO/Hangfire) ·
[Official website](https://www.hangfire.io) ·
[License: LGPL-3.0](https://github.com/HangfireIO/Hangfire/blob/main/LICENSE.md)

## Overview

Hangfire is a library for running background work — fire-and-forget, delayed, and recurring jobs — inside .NET applications, first released in 2013[^1]. Its defining design choice is that job state lives in a **persistent store you already run** (SQL Server by default, with Redis, PostgreSQL, and others via storage providers), rather than in memory or in a dedicated message broker. A call like `BackgroundJob.Enqueue(() => DoWork())` serializes the method invocation into a row in that store; a pool of worker threads polls the store and executes the job. Because the state is durable, jobs survive process restarts, deployments, and crashes, and the same mechanism gives you automatic retries and a web dashboard "for free."

The audience is server-side .NET teams — ASP.NET / ASP.NET Core apps, worker services, console hosts — who need reliable async work but don't want to stand up RabbitMQ, Kafka, or a separate scheduler tier. Hangfire's pitch is that your database is enough to get started, and it is: the barrier to a working setup is genuinely low.

The central tension is **at-least-once execution and storage coupling**. Jobs can and will run more than once (a worker can die after doing work but before marking the job complete), so job methods must be idempotent — a fact easy to miss until a job double-sends an email in production. And the "just use your database" convenience becomes a scaling ceiling: the default SQL Server storage polls for work, which caps throughput and latency in ways that push serious users toward Redis storage, which is a commercial add-on[^2].

## Getting Started

```
dotnet add package Hangfire
```

```csharp
// ASP.NET Core — Program.cs (minimal hosting model)
builder.Services.AddHangfire(cfg => cfg
    .UseSimpleAssemblyNameTypeSerializer()
    .UseRecommendedSerializerSettings()
    .UseSqlServerStorage(connectionString));   // creates schema on first run

builder.Services.AddHangfireServer();          // starts the worker pool

var app = builder.Build();
app.UseHangfireDashboard("/hangfire");          // web UI at /hangfire

// Enqueue work from anywhere — returns immediately.
BackgroundJob.Enqueue(() => Console.WriteLine("Runs on a worker thread"));

// Recurring job, cron-scheduled.
RecurringJob.AddOrUpdate("nightly-report", () => Reports.Generate(), Cron.Daily);
```

The expression `() => Reports.Generate()` is not a closure executed locally — Hangfire parses the expression tree, records the type, method, and serialized arguments, and reconstructs the call on a worker. This is why arguments must be serializable and why capturing large or non-serializable objects fails.

## Architecture / How It Works

Hangfire has three moving parts, decoupled through the storage layer:

1. **Client** — `BackgroundJob` / `RecurringJob`. Serializes the target method (type name, method, argument JSON) into the store and returns a job id. The client does no execution; it only writes state.
2. **Storage** — the source of truth. `Hangfire.SqlServer` (in-box), `Hangfire.Pro.Redis` (commercial), and community providers for PostgreSQL, MySQL, MongoDB, SQLite, etc. Storage defines the schema, distributed locks, and how workers fetch jobs.
3. **Server** (`BackgroundJobServer`) — a hosted component running a pool of worker threads plus recurring/scheduled-job dispatchers. Workers fetch a job and move it through a **state machine** (Enqueued → Processing → Succeeded / Failed / Scheduled / Deleted), persisting each transition.

Cross-cutting behavior is implemented as **job filters** (`IServerFilter`, `IElectStateFilter`), the same attribute-based interception model as ASP.NET MVC. The built-in `AutomaticRetryAttribute` is a filter: on an unhandled exception it re-schedules the job with exponential backoff, up to 10 attempts by default, after which the job lands in the **Failed** state and stays there for manual inspection or requeue from the dashboard.

Job fetching differs by storage. SQL Server storage uses **polling** — workers query for available jobs on an interval, using an "invisibility timeout" so a fetched-but-not-completed job becomes visible again if its worker dies. Redis storage uses blocking pops, which is both lower-latency and lower-load. This is the single most consequential internal difference between the free and paid paths.

Recurring jobs are stored as cron entries; a dispatcher wakes periodically, computes which are due, and enqueues concrete instances. There is no separate scheduler process — scheduling is just more work done by the same server against the same store.

## Production Notes

**Jobs must be idempotent.** Execution is at-least-once, not exactly-once. A worker that completes side effects and then crashes before persisting `Succeeded` will have the job retried. Guard external effects (emails, payments, webhooks) with idempotency keys or dedup checks.

**SQL Server polling is a latency/throughput floor.** Default `UseSqlServerStorage` polls, and enqueue-to-start latency is bounded by the poll interval. Set `SlidingInvisibilityTimeout` and `QueuePollInterval` deliberately. High-throughput deployments generally move to `Hangfire.Pro.Redis`, which is a paid license[^2] — budget for it if you expect thousands of jobs/minute.

**Dashboard authorization is a real footgun.** By default the dashboard only allows **local** requests; deployed behind a proxy, that check can pass for everyone or block everyone depending on how remote IPs resolve. You must supply an `IDashboardAuthorizationFilter` for any remote access — do not expose `/hangfire` publicly without one. Anyone with dashboard access can trigger, delete, and requeue jobs.

**Storage schema is versioned and migrates on startup.** `UseSqlServerStorage` runs schema migrations automatically; on least-privilege databases the app account needs DDL rights on first run, or you must apply the migration scripts out of band. Mixing Hangfire versions across servers that share one store can hit schema-compatibility errors — upgrade all servers together.

**Serialization is a compatibility surface.** Because jobs are stored as serialized method references, renaming a job method/type or changing argument shapes can orphan already-enqueued jobs (they fail to deserialize on execution). Prefer `UseRecommendedSerializerSettings()` and be cautious renaming anything that appears in an in-flight job.

**Concurrency control is opt-in.** Two servers sharing a queue will both pull work; nothing prevents the same *logical* task from running concurrently unless you add `[DisableConcurrentExecution]` or your own distributed lock. Long jobs that exceed the invisibility timeout can be picked up a second time while still running.

**In-process by default.** With `AddHangfireServer`, workers run inside the web app, coupling job capacity to web-tier scaling and meaning a web deploy interrupts running jobs (shutdown is graceful, but long jobs should be cancellation-aware). Many teams run a dedicated worker host instead.

## When to Use / When Not

**Use when:**
- You're on .NET and want durable background jobs without operating a broker.
- Your workload is moderate throughput and you already run SQL Server or Postgres.
- You value a built-in dashboard, automatic retries, and recurring/cron jobs out of the box.
- Jobs are, or can be made, idempotent.

**Avoid when:**
- You need very high throughput or sub-second dispatch latency on the free tier — SQL Server polling won't get there, and the low-latency path (Redis) is commercial.
- You want exactly-once semantics or a transactional outbox — Hangfire is at-least-once by design.
- You need a general message bus (pub/sub, fan-out, cross-service events) rather than in-app job execution.
- You're not on .NET.

## Alternatives

- dotnetcore/CAP — event-bus / outbox pattern for .NET with at-least-once delivery; use instead when you need reliable messaging between services, not just in-app jobs.
- quartznet/quartznet — mature scheduler with rich trigger/calendar semantics; use when scheduling complexity dominates and you don't need a job dashboard.
- jamesmh/coravel — lightweight in-memory scheduling/queuing; use when you want no persistent store or dashboard for a smaller app.
- MassTransit/MassTransit — full message bus over RabbitMQ / Azure Service Bus; use when you need broker-backed distributed messaging.
- sidekiq/sidekiq — the Redis-backed Ruby ancestor Hangfire is modeled on; relevant only outside .NET.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014-06 | First stable release; SQL Server storage, dashboard, recurring jobs. |
| 1.5 | 2015 | Job filters, refined storage abstraction, continuations. |
| 1.6 | 2016 | Batches (Pro), sliding invisibility timeout for SQL Server. |
| 1.7 | 2019 | .NET Core / ASP.NET Core first-class support (`AddHangfire`). |
| 1.8 | 2023 | Serialization improvements, dashboard and storage refinements. |

Early-version dates are approximate; treat the NuGet release feed as authoritative[^3]. The repository was created in 2013[^1] and remains actively maintained, with commits landing in 2026.

## References

[^1]: HangfireIO/Hangfire repository metadata (created 2013-08-06). https://github.com/HangfireIO/Hangfire
[^2]: Hangfire Pro — commercial Redis storage and Batches. https://www.hangfire.io/pro/
[^3]: Hangfire.Core on NuGet (release history). https://www.nuget.org/packages/Hangfire.Core/

## Tags

dotnet, csharp, background-jobs, job-queue, task-scheduler, cron, sql-server, redis, recurring-jobs, worker, distributed
