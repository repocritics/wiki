# bensheldon/good_job

> A multithreaded, Postgres-backed Active Job backend for Rails that trades raw throughput for transactional integrity and zero extra infrastructure.

[GitHub repo](https://github.com/bensheldon/good_job) ·
[Demo dashboard](https://goodjob-demo.herokuapp.com/) ·
[License: MIT](https://github.com/bensheldon/good_job/blob/main/LICENSE.txt)

## Overview

GoodJob is a background job backend for Ruby on Rails that stores its queue in
the application's own Postgres database rather than in Redis. It implements the
Active Job adapter interface, so jobs are ordinary `ActiveJob::Base` subclasses
and `perform_later` works unchanged; GoodJob only decides where the row lands and
which process picks it up. It was written by Ben Sheldon and released as 1.0 in
July 2020, explicitly inspired by Delayed::Job (record-based) and Que (advisory
locks)[^1].

The defining tradeoff is **integrity over throughput**. Because a job is a row in
your database, enqueuing it can participate in the surrounding transaction: if the
transaction rolls back, the job is never queued, and a job is never queued
referencing a record that was rolled back. Redis-backed queues like Sidekiq cannot
give this guarantee — the classic race where a worker dequeues a job before the
enqueuing transaction commits does not exist here. The cost is that Postgres, not a
purpose-built in-memory broker, becomes the throughput ceiling. GoodJob targets
"most workloads" — the README cites applications enqueuing on the order of one
million jobs/day — rather than the highest-volume Redis territory.

Two Postgres primitives do the heavy lifting. **Session-level advisory locks**
provide run-once safety without extra columns or a locking table, which keeps the
schema expressible in `schema.rb` (unlike Que, which needs `structure.sql` for its
triggers). **LISTEN/NOTIFY** wakes workers on new jobs so latency stays low without
tight polling. This is the same architectural family as Que, with a broader feature
set and a maintained web dashboard.

## Getting Started

```sh
bundle add good_job
bin/rails g good_job:install
bin/rails db:migrate
```

```ruby
# config/application.rb (or per-environment)
config.active_job.queue_adapter = :good_job
```

```ruby
# Enqueue like any Active Job — GoodJob supports queues, waits, priorities:
YourJob.set(queue: :default, wait: 5.minutes, priority: 10).perform_later
```

In development the adapter runs jobs `async` inside `rails server`; in test it runs
`inline`. In production the default is `external` — jobs are enqueued but executed by
a separate process:

```bash
bundle exec good_job start
```

```Procfile
web: rails server
worker: bundle exec good_job start
```

## Architecture / How It Works

A job is a row in the `good_jobs` table (later versions add `good_job_executions`,
`good_job_batches`, `good_job_processes`, and `good_job_settings`). A worker process
polls for and/or is notified of executable rows, takes a Postgres **session-level
advisory lock** on the row, runs the job, and — by default — deletes the row on
success. The advisory lock is what guarantees a job runs once even with many workers
racing for it; it is held for the life of the database session, not the transaction.

Concurrency is real OS-thread multithreading via `Concurrent::Ruby`, following
Rails' threading and code-execution guidelines[^2]. Each pool runs up to
`max_threads` (default 5) threads, and each active thread needs its own Postgres
connection. **LISTEN/NOTIFY** (enabled by default) pushes new-job notifications so
idle workers wake immediately; a poll interval (production default 10s) is a fallback
for LISTEN/NOTIFY blips and for future-scheduled jobs. Scheduled jobs are cached in
memory (up to `max_cache`, default 10k) so their execution time is known without
re-querying.

`execution_mode` decides the topology:

- `:inline` — run in the enqueuing process/thread. Test and development only.
- `:external` — enqueue only; a `good_job start` process executes. Production default.
- `:async` / `:async_server` — execute in threads inside the web server process
  (economical for small workloads; falls back to `:external` outside `rails server`).
- `:async_all` — execute in threads in any Rails process.

Beyond the queue itself, GoodJob ships features that Redis backends typically bolt on
separately: cron-style recurring jobs (`enable_cron`), job batches with callbacks,
concurrency and throttling controls (via advisory-lock keys), bulk enqueue, and a
mountable Rails engine web **Dashboard** for inspecting, retrying, and discarding
jobs. The dashboard is a Rack app inside your Rails router, so it inherits your app's
authentication rather than running as a separate service.

## Production Notes

- **Connection pool sizing is the first footgun.** Every working thread needs a
  Postgres connection, and LISTEN/NOTIFY holds a dedicated one. Your ActiveRecord
  pool must cover `max_threads` plus overhead, on top of web-server connections. Undersize
  it and workers block on checkout; oversize it and you exhaust Postgres `max_connections`.
- **PgBouncer transaction pooling breaks advisory locks.** Session-level advisory locks
  require a persistent session, which transaction-mode PgBouncer does not provide. Run
  GoodJob's connections through session-pooling mode (or a direct connection); the README
  documents the compatibility constraints. Getting this wrong produces jobs that appear to
  run twice or locks that never release.
- **Table bloat on high throughput.** Jobs are rows that are inserted and deleted
  constantly, generating dead tuples. Keep autovacuum aggressive on the GoodJob tables. If
  you set `preserve_job_records = true` (needed to keep the dashboard useful), the tables
  grow unbounded until you run `good_job cleanup_preserved_jobs` (default retains 14 days) —
  schedule it as a cron job or the table becomes the bottleneck.
- **Postgres is the throughput ceiling.** Each dequeue is a locking query. At sustained
  very-high job rates the polling/locking contention and `good_jobs` index churn become the
  limiting factor well before Redis-based backends would. Isolate hot queues into separate
  execution pools (`queues` supports comma-separated queues, `-` exclusion, and `;`-separated
  isolated pools) so a flood of small jobs cannot starve large ones.
- **Graceful shutdown.** `shutdown_timeout` defaults to `-1` (wait forever); a SIGKILL mid-job
  leaves the advisory lock to be reclaimed when the dead session's connection drops. Set a
  finite timeout under orchestrators that hard-kill pods, and understand that interrupted jobs
  rely on the session ending to be retried.
- **Major-version upgrades change the schema.** v1→v2, v2→v3, and v3→v4 each ship migrations;
  the README's per-version upgrade sections must be followed in order rather than skipped.

## When to Use / When Not

**Use when:**
- You already run Postgres and want to avoid operating Redis purely for jobs.
- Transactional enqueue matters — jobs must commit atomically with the data they act on.
- You want an integrated dashboard, cron, batches, and concurrency controls without extra gems.
- Your volume is moderate (thousands to low-millions of jobs/day) and correctness beats peak speed.

**Avoid when:**
- You push very high sustained throughput where a Redis broker's raw dequeue rate wins.
- You are not on Postgres, or need a genuinely database-agnostic backend.
- You cannot give GoodJob session-level Postgres connections (e.g. locked into transaction-mode PgBouncer).
- You want the framework-default path on a fresh Rails 8 app (that is now Solid Queue).

## Alternatives

- sidekiq/sidekiq — Redis-backed, highest proven throughput and mature ecosystem. Use instead when raw job volume dominates and you already operate Redis.
- rails/solid_queue — the Rails 8 default, database-backed and DB-agnostic. Use instead when you want the framework's blessed path or a non-Postgres database.
- que-rb/que — the other Postgres advisory-lock backend, minimal footprint. Use instead when you want the leanest option and accept `structure.sql`.
- collectiveidea/delayed_job — the original record-based backend, single-threaded. Use instead only for legacy simplicity or very low volume.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2020-07 | First stable release; advisory locks + LISTEN/NOTIFY, Active Job adapter[^1]. |
| 2.0 | 2021 | Execution model and schema refinements; dashboard maturation. |
| 3.0 | 2022 | Batches, cron, concurrency/throttling controls consolidated. |
| 4.0 | 2024 | Current major line; schema and configuration changes (see upgrade guide). |

(Minor releases ship frequently; exact 2.x–4.x dates should be confirmed against the
[releases page](https://github.com/bensheldon/good_job/releases) before citing.)

## References

[^1]: Ben Sheldon, "Introducing GoodJob 1.0" — island94.org, 2020-07. https://island94.org/2020/07/introducing-goodjob-1-0
[^2]: Rails Guides, "Threading and Code Execution in Rails." https://guides.rubyonrails.org/threading_and_code_execution.html

## Tags

ruby, ruby-on-rails, active-job, background-jobs, job-queue, postgres, multithreaded, advisory-locks, listen-notify, rails
