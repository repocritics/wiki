# hibiken/asynq

> A Redis-backed distributed task queue for Go — the de facto "Sidekiq for Go" for teams that already run Redis.

[GitHub repo](https://github.com/hibiken/asynq) ·
[License: MIT](https://github.com/hibiken/asynq/blob/master/LICENSE)

## Overview

Asynq is a Go library for enqueueing tasks and processing them asynchronously
with worker goroutines, backed entirely by Redis[^1]. A client marshals a task
(a type string plus an opaque `[]byte` payload) onto a queue; a server pulls
tasks off, dispatches each to a registered handler, and runs them concurrently.
It occupies the same niche in Go that Sidekiq occupies in Ruby or Celery in
Python: a batteries-included background-job system for people who do not want to
stand up Kafka or RabbitMQ but already have Redis in their stack.

The project was started by Ken Hibino in 2019[^1] and reached broad adoption on
the strength of its ergonomics: a `net/http`-style `ServeMux` for routing task
types to handlers, weighted and strict priority queues, at-least-once delivery,
scheduled and periodic tasks, unique/deduplicated tasks, per-task timeouts and
deadlines, and a companion web UI and CLI. Its defining tension is maturity
versus governance: the code is stable and widely deployed, yet it has remained
on a `v0.x` version line for its entire life[^2], and maintainer activity has
slowed considerably, leaving a large backlog of open issues and pull requests.
As of 2026 it carries roughly 13.5k stars and ~960 forks but should be read as a
proven-but-quiet dependency rather than a fast-moving one.

## Getting Started

Requires Go (last two releases supported) and a Redis server, version 4.0 or
higher[^1].

```sh
go get -u github.com/hibiken/asynq
```

```go
// producer — enqueue a task
client := asynq.NewClient(asynq.RedisClientOpt{Addr: "127.0.0.1:6379"})
defer client.Close()

payload, _ := json.Marshal(map[string]int{"user_id": 42})
task := asynq.NewTask("email:welcome", payload)
info, err := client.Enqueue(task, asynq.MaxRetry(5), asynq.Queue("critical"))
// info.ID, info.Queue identify the enqueued task
```

```go
// worker — process tasks
srv := asynq.NewServer(
    asynq.RedisClientOpt{Addr: "127.0.0.1:6379"},
    asynq.Config{
        Concurrency: 10,
        Queues:      map[string]int{"critical": 6, "default": 3, "low": 1},
    },
)

mux := asynq.NewServeMux()
mux.HandleFunc("email:welcome", func(ctx context.Context, t *asynq.Task) error {
    // return asynq.SkipRetry to fail permanently without retrying
    return sendEmail(t.Payload())
})

if err := srv.Run(mux); err != nil {
    log.Fatal(err)
}
```

## Architecture / How It Works

Everything is stored in Redis. Each queue is a set of Redis data structures —
lists for pending tasks, sorted sets keyed by timestamp for scheduled, retry,
and "dead"/archived tasks — and state transitions are performed by Lua scripts
so that dequeue, retry-scheduling, and lease renewal are atomic against the
server[^1]. A task moves through pending → active → (completed | retry |
archived) states, and the server tracks in-flight tasks with a lease that must
be periodically renewed.

Delivery is **at-least-once**, not exactly-once. A task is only removed from the
active set after the handler returns success; if a worker crashes mid-task, the
lease expires and the task is recovered and re-run[^1]. Handlers must therefore
be idempotent. Retries are automatic with exponential backoff by default, up to
`MaxRetry`, after which the task is archived rather than deleted so it can be
inspected or re-run.

Queue selection is weighted by default: with `{"critical": 6, "default": 3,
"low": 1}` the server dequeues from each queue proportionally to its weight, so
low-priority work is never fully starved. A `StrictPriority` option changes this
to absolute ordering. Concurrency is a fixed goroutine budget shared across all
queues. Routing is done through `ServeMux`, which matches task-type strings the
way `http.ServeMux` matches URL paths and supports middleware wrapping.

The broader system is three repositories: the core library (`asynq`), the web UI
`hibiken/asynqmon`, and the `tools/asynq` CLI. The web UI and CLI both talk to
the same Redis keys the library uses, so operational tooling requires no extra
service — but it also means the Redis key schema is effectively a public
contract, and Prometheus metrics are exported for queue depth and latency.

## Production Notes

- **It is still `v0.x`.** The maintainers state plainly that the public API can
  change without a major-version bump before a `v1.0.0` that has not shipped
  after years of use[^2]. Pin exact versions and read release notes before
  upgrading; do not assume semver guarantees.
- **Maintenance has slowed.** Issue and PR throughput has dropped and the open
  backlog is large (a few hundred open issues as of 2026). The code works, but
  expect delays on upstream fixes and plan to vendor or patch if you hit a bug.
- **Redis Cluster is only partially supported.** Some of the Lua scripts are not
  compatible with Redis Cluster[^1]. If you run Cluster, test thoroughly; many
  users run a single Redis primary with Sentinel for failover instead.
- **Redis is a single point of contention and durability boundary.** Throughput
  and task durability are bounded by your Redis instance and its persistence
  settings. If Redis loses data (e.g. AOF disabled and a crash), enqueued tasks
  are lost — Asynq inherits Redis's durability, it does not add its own.
- **Payloads are opaque bytes** with no built-in schema or versioning. Changing
  a payload struct can break tasks still queued from an older deploy — version
  payloads explicitly.
- **At-least-once means duplicates happen.** Crashes, timeouts, and lease expiry
  re-run tasks. Make handlers idempotent; the unique-task option is coarse dedup,
  not exactly-once. Poison tasks churn through retries into the archive, so set
  realistic `Timeout`/`Deadline` per type and monitor the archived set.

## When to Use / When Not

**Use when:**
- You already run Redis and want background jobs in Go without new infrastructure.
- You need priority queues, scheduling, retries, and periodic (cron) tasks out
  of the box with a clean handler interface.
- You want operational visibility (web UI, CLI, Prometheus) for free.
- At-least-once delivery with idempotent handlers fits your workload.

**Avoid when:**
- You need transactional job enqueue tied to your primary database — a
  Postgres-backed queue keeps jobs and business data in one commit.
- You require exactly-once semantics or strong durability beyond what Redis
  gives you.
- You want a dependency with an active maintainer and a stable `v1.x` API
  guarantee — Asynq is neither fast-moving nor formally stabilized.
- You run Redis Cluster as a hard requirement.

## Alternatives

- riverqueue/river — Postgres-backed Go job queue; enqueue jobs in the same
  transaction as your data. Use when you want transactional guarantees over Redis speed.
- vmihailenco/taskq — Go task queue with pluggable backends (Redis, SQS, IronMQ).
  Use when you need brokers other than Redis.
- RichardKnop/machinery — older Go distributed task queue supporting AMQP, Redis,
  and more. Use when you need AMQP/RabbitMQ as the broker.
- contribsys/faktory — language-agnostic job server by Sidekiq's author. Use when
  jobs are produced/consumed across multiple languages, not just Go.
- hatchet-dev/hatchet — Postgres-backed durable task orchestration with DAGs. Use
  when you need workflow orchestration, not just a flat queue.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-11 | Repository created by Ken Hibino[^1]. |
| 0.x | 2020–2021 | Core queue, priority queues, scheduler, retries, CLI, `asynqmon` web UI. |
| 0.x | 2021–2022 | Task aggregation/groups, unique tasks, periodic tasks, Prometheus metrics. |
| 0.x | 2023–2026 | Continued 0.x line; slower cadence, no `v1.0.0` release[^2]. |

## References

[^1]: Asynq README — features, architecture overview, Redis 4.0+ requirement, Redis Cluster caveat. https://github.com/hibiken/asynq/blob/master/README.md
[^2]: Asynq README, "Stability and Compatibility" — states the project remains on `v0.x.x` and the public API may change without a major-version bump before `v1.0.0`. https://github.com/hibiken/asynq#stability-and-compatibility

## Tags

go, golang, task-queue, background-jobs, redis, distributed-systems, worker-pool, job-scheduler, asynchronous, at-least-once
