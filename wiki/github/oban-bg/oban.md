# oban-bg/oban

> Database-backed background job processing for Elixir, using a Postgres (or MySQL/SQLite) table as the queue instead of Redis.

[GitHub repo](https://github.com/oban-bg/oban) ·
[Official website](https://oban.pro) ·
[License: Apache-2.0](https://github.com/oban-bg/oban/blob/main/LICENSE.txt)

## Overview

Oban is a job-queue library for Elixir that stores jobs as rows in your
application's SQL database rather than in a dedicated broker like Redis or
RabbitMQ. It was created by Parker Selbert (originally under the `sorentwo`
account, now the `oban-bg` organization) and first appeared in 2019[^1]. The
core proposition is that if you already run PostgreSQL, you already have
everything a durable queue needs: transactions, backups, and a place to keep
history.

The defining design choice is that jobs are *not deleted* after they run —
completed and failed rows are retained in the `oban_jobs` table for inspection
and metrics. This is what makes Oban observable, and it is also the source of
its most common operational surprise: the table grows without bound unless you
actively prune it. Fetching uses PostgreSQL's `SELECT ... FOR UPDATE SKIP
LOCKED`, so many nodes can pull work concurrently without lock contention[^2].

Oban is open-core. The library in this repository (Apache-2.0) is a complete,
production-usable queue on its own. A separate commercial tier, Oban Pro, and a
LiveView dashboard, Oban Web, are sold under license and gate the genuinely
distributed features — global concurrency, global rate limiting, workflows, and
batches[^3]. Understanding where the free/paid line falls is the single most
important thing to know before adopting it.

## Getting Started

Add the dependency and run the generated migration, then configure a repo and
queues:

```elixir
# mix.exs
def deps do
  [{:oban, "~> 2.0"}]
end
```

```elixir
# config/config.exs
config :my_app, Oban,
  repo: MyApp.Repo,
  queues: [mailers: 20, default: 10]
```

```elixir
# A worker
defmodule MyApp.MailerWorker do
  use Oban.Worker, queue: :mailers, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"email" => email}}) do
    MyApp.Email.deliver(email)
    :ok
  end
end

# Enqueue — can share a transaction with other writes via Oban.insert/2
%{email: "foo@example.com"}
|> MyApp.MailerWorker.new()
|> Oban.insert()
```

## Architecture / How It Works

Oban runs as a supervision tree under your application. Each configured queue is
a producer that periodically polls the database for available jobs and pulls a
batch using `FOR UPDATE SKIP LOCKED`. Every job executes in its own supervised
process, so a crash cleans up in isolation and cannot take down the queue[^2].

To avoid pure polling latency, Oban uses a *notifier*. The default on
PostgreSQL is `LISTEN`/`NOTIFY`: inserting a job fires a `pg_notify` so idle
nodes wake immediately rather than waiting for the next poll tick. There is also
a `Oban.Notifiers.PG` notifier built on distributed Erlang for environments
where `LISTEN`/`NOTIFY` is unavailable.

Since version 2.0 the persistence layer is pluggable via *engines*. The Basic
engine drives PostgreSQL; MySQL and SQLite3 engines also ship, though they are
newer and less battle-tested than the Postgres path[^4]. Cross-cutting behavior
(pruning old jobs, rescuing orphans, running cron schedules) is implemented as
*plugins* rather than baked into the core loop.

Schema evolution is handled through versioned Ecto migrations. Oban ships an
`Oban.Migration` module and bumps a schema version (v11, v12, …) as internal
structure changes; upgrading across major releases usually means generating and
running a new migration.

The commercial Smart Engine (Pro) replaces the Basic engine to add global
concurrency limits, global rate limiting, and queue partitioning — coordination
that the open engine deliberately does not attempt.

## Production Notes

- **The jobs table grows forever by default.** Retention is the whole point, but
  you must run the `Oban.Plugins.Pruner` (or Pro's `DynamicPruner`) to bound
  `oban_jobs`. Teams that skip this discover a multi-gigabyte table and slow
  inserts months later.
- **PgBouncer in transaction-pooling mode breaks `LISTEN`/`NOTIFY`.** Session
  pooling is required for the default notifier, or you must switch to the PG
  (distributed-Erlang) notifier. This is the most-reported deployment footgun.
- **Global concurrency is not in the free tier.** The open engine enforces
  concurrency *per node*. Ten nodes with `queue: [x: 5]` run up to 50 jobs at
  once. Capping total cluster concurrency or rate-limiting an external API
  requires Pro's Smart Engine.
- **Long jobs and shutdown.** Queues drain within a grace period on shutdown;
  jobs still running past it are left in an executing state and rely on the
  `Rescuer` plugin (or Pro equivalent) to be picked back up.
- **Unique jobs are best-effort under races.** Uniqueness is enforced with
  advisory locks and query-time checks, not a database constraint; tight insert
  races have historically been able to slip a duplicate through.
- **Queue backpressure hits Postgres.** Because the queue *is* the database, a
  flood of jobs, aggressive polling intervals, or very short job durations put
  load directly on your primary. High-throughput deployments tune poll
  intervals and consider a dedicated database.

## When to Use / When Not

**Use when:**
- You run Elixir on top of PostgreSQL and want jobs enqueued transactionally
  alongside your other writes.
- You want durable jobs, retries, cron scheduling, and history without
  introducing Redis or a separate broker.
- Operational simplicity (one datastore, one backup) matters more than raw
  throughput ceiling.

**Avoid when:**
- You need global concurrency or cross-cluster rate limiting and won't pay for
  Oban Pro.
- Your workload is extreme-throughput, fire-and-forget with no durability need —
  a Redis-backed or in-memory queue will push more jobs per second.
- You aren't on Elixir, or you're on MySQL/SQLite and want the most mature,
  most-exercised engine (that is Postgres).

## Alternatives

- riverqueue/river — Go queue with the same Postgres `SKIP LOCKED` design; use
  when your services are in Go rather than Elixir.
- timgit/pg-boss — Node.js Postgres-backed queue; same "database is the broker"
  philosophy for JavaScript stacks.
- akira/exq — Redis-backed, Sidekiq-compatible Elixir queue; use when you want
  Redis throughput and don't need transactional enqueue.
- quantum-elixir/quantum-core — pure cron scheduling for Elixir; use when you
  only need periodic triggers, not durable job state.
- sidekiq/sidekiq — the Ruby reference point; use when you're in Ruby and
  already run Redis.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-02 | Initial development under `sorentwo`[^1]. |
| 1.0 | 2019 | First stable release; PostgreSQL-only, advisory-lock fetching. |
| 2.0 | 2020-09 | Pluggable engines and notifiers; plugin architecture[^4]. |
| 2.x | 2021–2026 | MySQL and SQLite3 engines added; ongoing 2.x line[^4]. |

## References

[^1]: Oban repository, created 2019-02-25 under the `oban-bg` organization (formerly `sorentwo/oban`). https://github.com/oban-bg/oban
[^2]: Oban documentation — job execution, `FOR UPDATE SKIP LOCKED`, and per-job process isolation. https://oban.hexdocs.pm/Oban.html
[^3]: Oban Pro and Oban Web are commercial licensed extensions (Smart Engine, Workflows, Batches, dashboard). https://oban.pro
[^4]: Oban changelog and engine documentation (PostgreSQL, MySQL, SQLite3 engines; 2.0 architecture). https://oban.hexdocs.pm/changelog.html

## Tags

elixir, background-jobs, job-queue, postgresql, ecto, task-queue, open-core, beam, cron, mysql, sqlite
