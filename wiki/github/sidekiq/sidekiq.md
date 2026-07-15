# sidekiq/sidekiq

> Redis-backed, thread-based background job processing for Ruby — the default choice for Rails job queues since the mid-2010s.

[GitHub repo](https://github.com/sidekiq/sidekiq) ·
[Official website](https://sidekiq.org) ·
[License: LGPL-3.0](https://github.com/sidekiq/sidekiq/blob/main/LICENSE.txt)

## Overview

Sidekiq is a background job system for Ruby, created by Mike Perham and first released in 2012[^1]. Its defining design decision was to run many jobs concurrently as **threads inside a single process**, rather than forking a process per job as its predecessor Resque did. On memory-constrained Ruby deployments this was a large practical win: one Sidekiq process with a pool of worker threads replaces dozens of forked Resque workers, cutting RAM use substantially. That tradeoff — threads over processes — is the whole story, and it is also the source of most of its footguns.

Sidekiq stores queues, retries, scheduled jobs, and dead jobs in **Redis** (or a compatible server — the README lists Valkey 7.2+ and Dragonfly 1.27+, with Redis 7.2.4 treated as the canonical target[^2]). Jobs are plain JSON payloads pushed onto Redis lists; workers pull them with a blocking pop. There is no separate broker, database schema, or coordinator — Redis is the single source of truth, which is both the simplicity and the durability ceiling of the system.

The project is **dual-licensed and commercially funded**. The open-source core is LGPL-3.0 (unusual for a Ruby gem, where MIT dominates), and Perham's company Contributed Systems sells two proprietary tiers, Sidekiq Pro and Sidekiq Enterprise, which add reliability, batches, rate limiting, and unique jobs[^3]. Several features people assume are "in Sidekiq" are actually paid — most importantly, reliable job fetching. This matters when reasoning about data-loss guarantees (see Production Notes).

## Getting Started

```bash
bundle add sidekiq
```

Define a job and enqueue it:

```ruby
# app/jobs/hard_worker.rb
class HardWorker
  include Sidekiq::Job
  sidekiq_options queue: :default, retry: 5

  def perform(user_id, note)
    User.find(user_id).notify!(note)
  end
end

# enqueue asynchronously
HardWorker.perform_async(42, "welcome")

# enqueue with a delay
HardWorker.perform_in(1.hour, 42, "reminder")
```

Run the worker process (separate from your web server):

```bash
bundle exec sidekiq -q critical -q default -c 10
```

With Rails, `HardWorker` can instead be an `ActiveJob` subclass and Sidekiq will act as the queue adapter — at a measurable CPU cost (see below).

## Architecture / How It Works

The system has three cooperating parts, all coordinating only through Redis:

1. **Client** — `perform_async` serializes the job class name and arguments to JSON and `LPUSH`es it onto a Redis list named after the queue. This is the only thing your web process does; it never touches a thread pool.
2. **Server (the `sidekiq` process)** — a pool of worker threads runs a blocking `BRPOP` across the queues it was told to watch, deserializes the payload, and calls `perform`. A **Capsule** (introduced in Sidekiq 7) is a self-contained thread pool with its own concurrency and queue list, so one process can host multiple independent pools[^4].
3. **Redis data structures** — queues are lists; the retry set and scheduled set are sorted sets keyed by execution time; failed-out jobs land in the "dead set" (historically "morgue"). A poller promotes scheduled/retry jobs into their live queue when their time arrives.

**Middleware chains.** Both client and server expose an ordered middleware stack, which is how nearly every add-on (metrics, unique jobs, batches, APM instrumentation) hooks in. Understanding the chain is the key to extending Sidekiq without patching it.

**Retries.** By default a failed job is retried with exponential backoff up to a fixed number of attempts, then moved to the dead set for manual inspection. Because delivery is **at-least-once**, a job can run more than once (crash after side effect, before ack), so jobs must be written to be idempotent. This is the single most important correctness constraint in the system.

**Arguments must be simple.** Job arguments are serialized to JSON, so only strings, numbers, booleans, arrays, and hashes survive the round trip. Passing an ActiveRecord object "works" only because it silently serializes to something lossy; the idiom is to pass an ID and re-load. ActiveJob's GlobalID papers over this but adds deserialization overhead.

**The GVL constraint.** On MRI/CRuby, the Global VM Lock means threads do not execute Ruby bytecode in parallel. Sidekiq's concurrency is therefore only a win for **I/O-bound** work (network calls, DB queries, Redis) where threads release the GVL while waiting. CPU-bound jobs do not scale across threads and will serialize; those need multiple processes (or JRuby, which has no GVL).

## Production Notes

**Basic Sidekiq can lose in-flight jobs.** Open-source Sidekiq uses `BRPOP`, which removes a job from Redis the instant a worker picks it up. If that worker process is killed hard (OOM, `kill -9`, spot-instance reclaim) while the job is mid-flight, the job is gone — it was neither completed nor returned to the queue. Reliable fetch (`super_fetch`), which keeps an in-progress copy until the job acks, is a **Sidekiq Pro** feature[^3]. Teams that need at-least-once durability across crashes on the free tier must design around this (idempotency plus external reconciliation) or pay for Pro.

**Concurrency is not "set it high."** The README notes real applications rarely need concurrency above 10, and default concurrency in recent versions is deliberately small. Each thread holds a database connection, so the worker's concurrency must not exceed your ActiveRecord connection pool size, or threads will block waiting for connections. This mismatch is a classic Sidekiq production bug.

**Redis is a hard dependency and a SPOF.** All queue state lives in Redis; if Redis is down, nothing enqueues or runs. Size Redis memory for your worst-case backlog (a retry storm can balloon the sorted sets), enable persistence appropriate to your data-loss tolerance, and monitor memory. A Redis failover mid-job means those jobs follow the loss semantics above.

**ActiveJob adds overhead.** The project's own benchmark shows the `ActiveJob` adapter processing the same workload at roughly half the throughput of the native `Sidekiq::Job` API, due to argument (de)serialization and callback machinery[^2]. Native API also unlocks features (batches, native `sidekiq_options`) that ActiveJob cannot express. Use ActiveJob for portability; use `Sidekiq::Job` when throughput or advanced features matter.

**Long-running jobs fight deploys.** On shutdown Sidekiq gives running jobs a timeout to finish, then pushes the survivors back for retry. Jobs longer than that window get interrupted and re-run — another reason idempotency is mandatory, and a reason to keep jobs short and chunked.

**Web UI is a mountable Rack app** but exposes queue contents and controls; mount it behind authentication. It is Redis-backed and reflects live state, so it is safe to run multiple copies.

**Upgrades** are documented per major version in the repo's `docs/` directory, and `bundle up sidekiq` is the recommended path. Major versions raise the Redis and Ruby floors (8.0 requires Redis 7.0+ and Ruby 3.2+/JRuby 9.4+[^2]); check those before upgrading production.

## When to Use / When Not

**Use when:**
- You already run Redis and want the highest-throughput, lowest-ceremony Ruby job system.
- Your jobs are I/O-bound (emails, HTTP calls, DB writes) — exactly where thread concurrency pays off.
- You're on Rails and want the de facto standard with the deepest ecosystem and documentation.
- You can make jobs idempotent and are comfortable with at-least-once semantics.

**Avoid when:**
- You cannot run or don't want to operate a Redis server — a Postgres-backed queue keeps jobs in your existing database.
- You need transactional enqueue (job committed atomically with your DB change); Redis can't join a Postgres transaction, so an enqueue can succeed while its transaction rolls back.
- Your jobs are CPU-bound on MRI — the GVL will serialize them; you'll need processes, not threads.
- You need guaranteed no-loss delivery for free — that's a Pro feature, not the open-source core.

## Alternatives

- resque/resque — the forking predecessor; process-per-job gives memory isolation between jobs at much higher RAM cost. Use when a leaky or crash-prone job must not affect its neighbors.
- bensheldon/good_job — Postgres-backed, ActiveJob-native, multithreaded. Use when you want one datastore and transactional enqueue without adding Redis.
- que-rb/que — Postgres-backed using advisory locks; very low latency, transactional. Use when jobs must commit atomically with your data changes.
- collectiveidea/delayed_job — the classic ActiveRecord-backed queue. Use for small apps that want zero new infrastructure and don't need high throughput.
- Solid Queue (Rails default from 7.1+) — database-backed, bundled with modern Rails. Use when you want the framework's batteries-included default rather than an external service.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012 | Initial release; threads-in-one-process model[^1]. |
| 4.0 | 2016 | Core rewrite for throughput and lower Redis load. |
| 5.0 | 2017 | Rails 5 / ActiveJob integration improvements. |
| 6.0 | 2019 | Lifecycle events, raised Ruby/Redis floors. |
| 7.0 | 2022-10 | Capsules, embedded mode, built-in metrics dashboard[^4]. |
| 8.0 | 2025 | Redis 7.0+ / Ruby 3.2+ / JRuby 9.4+, Rails & ActiveJob 7.0+[^2]. |

## References

[^1]: Mike Perham, "Sidekiq" — original release, 2012. https://www.mikeperham.com/2012/02/13/sidekiq-simple-efficient-background-processing-for-ruby/
[^2]: Sidekiq README — requirements, performance benchmark, and version support. https://github.com/sidekiq/sidekiq/blob/main/README.md
[^3]: Sidekiq Pro and Enterprise feature comparison (reliability, batches, rate limiting, unique jobs). https://sidekiq.org/products/pro.html
[^4]: Sidekiq 7.0 release notes — Capsules, embedded mode, metrics. https://www.mikeperham.com/2022/10/27/sidekiq-7.0/

## Tags

ruby, rails, background-jobs, job-queue, redis, concurrency, threading, activejob, worker, infrastructure
