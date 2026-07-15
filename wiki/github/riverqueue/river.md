# riverqueue/river

> A Postgres-backed background job queue for Go, built around transactional enqueueing.

[GitHub repo](https://github.com/riverqueue/river) ·
[Official website](https://riverqueue.com) ·
[License: MPL-2.0](https://github.com/riverqueue/river/blob/master/LICENSE)

## Overview

River is a background job processing system for Go that uses Postgres as its
only backing store[^1]. It was created by Brandur Leach and Blake Gentry and
first released publicly in late 2023[^2]. Its defining design choice is
transactional enqueueing: because jobs live in the same database as your
application data, you can insert a job inside the same transaction that writes
your business rows. If that transaction commits, the job is guaranteed to be
enqueued; if it rolls back, the job never existed and is never worked[^3]. This
eliminates a whole category of dual-write bugs common to Redis- or broker-based
queues, where the job store and the database can disagree after a partial
failure.

The tradeoff is equally direct: River puts your queue load on your primary
Postgres database. For teams already committed to Postgres, this is a feature —
one system to operate, back up, and reason about. For teams that treat the
database as a scarce shared resource, it is a real constraint, since job
throughput is bounded by Postgres write and vacuum capacity. River is in the
same lineage as Oban (Elixir), GoodJob (Ruby), and que — "use the database you
already have" — brought to Go with generics-based, type-safe job definitions[^1].

River is developed alongside a commercial offering (River Pro / River Cloud);
the core library in this repository is open source under MPL-2.0. As of this
writing it is actively maintained, with frequent commits and a companion web UI
in a separate repository (riverqueue/riverui).

## Getting Started

```bash
go get github.com/riverqueue/river
go get github.com/riverqueue/river/riverdriver/riverpgxv5
# Run schema migrations (creates river_job and related tables):
go run github.com/riverqueue/river/cmd/river migrate-up --database-url "$DATABASE_URL"
```

```go
// A job is a struct pair: JobArgs (the data) + Worker (the behavior).
type SortArgs struct {
    Strings []string `json:"strings"`
}

func (SortArgs) Kind() string { return "sort" }

type SortWorker struct {
    river.WorkerDefaults[SortArgs]
}

func (w *SortWorker) Work(ctx context.Context, job *river.Job[SortArgs]) error {
    sort.Strings(job.Args.Strings)
    return nil
}

// Wire up and start.
workers := river.NewWorkers()
river.AddWorker(workers, &SortWorker{})

client, err := river.NewClient(riverpgxv5.New(dbPool), &river.Config{
    Queues:  map[string]river.QueueConfig{river.QueueDefault: {MaxWorkers: 100}},
    Workers: workers,
})
// Enqueue transactionally alongside your own writes:
_, err = client.InsertTx(ctx, tx, SortArgs{Strings: []string{"whale", "bear"}}, nil)
```

## Architecture / How It Works

Jobs are rows in a `river_job` table. Each job has a `kind` (a stable string
that maps args to a worker), a JSONB `args` payload, a state, an attempt count,
and scheduling metadata. Workers are registered at startup so River can route a
fetched row's `kind` to the correct `Work` function.

Job fetching uses Postgres `SELECT ... FOR UPDATE SKIP LOCKED`, the standard
pattern for a database-backed queue: each worker grabs a batch of available
rows, skipping rows already locked by other workers, so many worker processes
can pull from the same table without contending on a single lock[^3]. To avoid
constant polling latency, River uses Postgres `LISTEN`/`NOTIFY` to wake workers
the moment a job is inserted, falling back to periodic polling as a safety net.

River runs a set of background **maintenance services** inside the client
process[^4]: a scheduler that promotes `scheduled`/`retryable` jobs to
`available` when their time arrives, a periodic-job enqueuer for cron-style
recurring work, a rescuer that recovers jobs orphaned by a crashed worker, a
cleaner that removes old completed/cancelled jobs, and a reindexer. Jobs move
through an explicit state machine (`available`, `running`, `retryable`,
`scheduled`, `completed`, `cancelled`, `discarded`, `pending`).

Database access is abstracted behind a **driver** interface. The primary driver
is `riverpgxv5` (pgx v5); a `database/sql` driver also exists so River can share
a pool with code that doesn't use pgx[^5]. This driver seam is also how River
supports inserting jobs from non-Go languages (Python, Ruby) that are then
worked by Go processes — insertion is just a SQL write against the same schema.

## Production Notes

**It runs on your primary database.** Every insert, fetch, state transition, and
cleanup is Postgres traffic. High-churn job tables bloat quickly; `river_job` is
subject to heavy `UPDATE`/`DELETE` load, so autovacuum tuning and monitoring
table bloat matter at scale. The maintenance cleaner removes finished jobs, but
retention windows and vacuum settings are yours to tune.

**LISTEN/NOTIFY and connection poolers.** River's low-latency notification path
depends on `LISTEN`/`NOTIFY`. PgBouncer in transaction-pooling mode does not
support session-level `LISTEN`, so behind such a pooler River degrades to
polling — jobs still run, but pickup latency rises. Route River's notifier
connection around the pooler (or use session pooling) if low latency matters.

**Throughput ceiling.** River is well suited to low-to-moderate and bursty
throughput. For sustained very high volumes (tens of thousands of jobs per
second), a purpose-built broker (Redis, NATS, Kafka) will out-scale a Postgres
table. Benchmark against your own database before assuming River is or isn't the
bottleneck.

**Migrations are versioned.** River ships its schema as numbered migrations run
via the `river` CLI or a Go migration API. Upgrading the library sometimes
introduces a new schema version; plan migration steps into deploys rather than
assuming a drop-in library bump.

**Shutdown.** Graceful stop drains in-flight work up to `SoftStopTimeout`, then
cancels via context. Long-running jobs must respect `ctx` cancellation or they
will be force-stopped; jobs that ignore context can block a clean shutdown.

**Generics.** River leans on Go generics (`river.Job[T]`, `WorkerDefaults[T]`),
so it requires a reasonably modern Go toolchain and gives compile-time type
safety on job payloads.

## When to Use / When Not

**Use when:**
- You already run Postgres and want jobs enqueued transactionally with your data.
- You want exactly-once-ish semantics tied to a DB commit, not a best-effort
  push to a separate broker.
- Your throughput is low-to-moderate and you value one fewer system to operate.
- You're in Go and want type-safe, generics-based job definitions.

**Avoid when:**
- You need to keep queue load off your primary database.
- You require sustained very high throughput better served by a dedicated broker.
- Your workers are polyglot: River inserts cross-language but only *works* jobs
  in Go.
- You aren't on Postgres and don't want to adopt it just for a queue.

## Alternatives

- hibiken/asynq — Redis-backed Go job queue; use it when you'd rather put queue
  load on Redis than on Postgres.
- vgarvardt/gue — older, lighter Postgres-backed Go queue; use it when you want
  the same DB-as-queue model with a smaller surface and no commercial tier.
- contribsys/faktory — language-agnostic job server; use it when workers are
  written in several languages, not just Go.
- bensheldon/good_job — Postgres-backed queue for Ruby/Rails; use it when your
  app is Rails rather than Go (a conceptual sibling of River).
- sorentwo/oban — Postgres-backed queue for Elixir; the design River was
  inspired by, for teams on the BEAM.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2023-11-08 | Public repository opened; initial 0.x development[^2]. |
| 0.x series | 2023–2026 | Iterative pre-1.0 releases: drivers, unique jobs, batch insert (COPY FROM), periodic jobs, web UI, cross-language insertion[^1]. |

## References

[^1]: River README and feature list. https://github.com/riverqueue/river
[^2]: Repository metadata, riverqueue/river (created 2023-11-08). https://github.com/riverqueue/river
[^3]: "Transactional enqueueing," River docs. https://riverqueue.com/docs/transactional-enqueueing
[^4]: "Maintenance services," River docs. https://riverqueue.com/docs/maintenance-services
[^5]: "Database drivers," River docs. https://riverqueue.com/docs/database-drivers

## Tags

go, golang, background-jobs, job-queue, postgres, postgresql, queue, transactional, worker-pool, task-scheduling
