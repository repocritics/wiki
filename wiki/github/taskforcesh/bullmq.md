# taskforcesh/bullmq

> A Redis-backed job and message queue for Node.js (and, increasingly, other runtimes) built on atomic Lua scripts.

[GitHub repo](https://github.com/taskforcesh/bullmq) ·
[Official website](https://bullmq.io) ·
[License: MIT](https://github.com/taskforcesh/bullmq/blob/master/LICENSE.md)

## Overview

BullMQ is the TypeScript rewrite of Bull, the long-standing Redis-based queue library for Node.js[^1]. Both come from the same author (Manuel Astudillo / OptimalBits, now taskforce.sh). Bull (v3) is in maintenance; BullMQ is where all new development happens. The value proposition is narrow and honest: if you already run Redis and need durable background jobs — delayed execution, retries with backoff, rate limiting, concurrency control, priorities, cron-style repeats, parent/child flows — BullMQ gives you that without standing up a dedicated queue broker.

The defining design choice is that all queue state lives in Redis and all state transitions are implemented as **Lua scripts executed server-side**, which makes each move (add, fetch-next, complete, fail, retry, promote-delayed) atomic under concurrency. This is what lets many workers pull from the same queue without double-processing. It also means BullMQ's correctness is bounded by Redis's own durability guarantees — a queue is exactly as persistent as the Redis instance behind it, and "at-least-once" delivery is the contract, not "exactly-once."

BullMQ is a library, not a service. There is no server process, no web UI, and no built-in observability beyond events. The maintainers monetize through BullMQ Pro (adds groups, batches, and observables) and Taskforce.sh (a hosted dashboard)[^2]; the open-source core is MIT and fully usable on its own. As of 2026 the project also ships native ports for Python, Rust, Elixir, and PHP in the same repo, though the Node.js implementation remains the reference and the most complete.

## Getting Started

```bash
npm install bullmq
# requires a reachable Redis (>= 6.2 recommended) or a compatible server (Dragonfly, Valkey)
```

```ts
import { Queue, Worker, QueueEvents } from 'bullmq';

const connection = { host: '127.0.0.1', port: 6379 };

// Producer
const queue = new Queue('paint', { connection });
await queue.add('car', { color: 'blue' }, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 1000 },
});

// Consumer — concurrency and rate limit are per-Worker
const worker = new Worker('paint', async job => {
  await paintCar(job.data.color);
}, { connection, concurrency: 5, limiter: { max: 100, duration: 1000 } });

// Cross-process completion events (backed by a Redis stream)
const events = new QueueEvents('paint', { connection });
events.on('completed', ({ jobId }) => console.log('done', jobId));
events.on('failed', ({ jobId, failedReason }) => console.error(jobId, failedReason));
```

## Architecture / How It Works

**Data model.** A queue is a set of Redis keys under a shared prefix. Jobs live in a hash; their ids are threaded through several structures by state: a `wait` list (LIST), a `prioritized` set (ZSET), `delayed` (ZSET keyed by timestamp), `active` (LIST), `completed`/`failed` (ZSETs), plus `waiting-children` for flows. Moving a job between states is never done client-side — the client invokes a Lua script that reads and mutates all relevant keys in one atomic Redis call.

**Workers.** `Worker` runs a blocking `BRPOPLPUSH`/`BLMOVE`-style loop to atomically move a job from `wait` to `active`, executes your async processor, then runs another script to move it to `completed` or `failed`. Because the fetch is atomic, N workers on one queue is the standard horizontal-scaling path. `concurrency` controls how many jobs one worker processes in parallel within a single Node process.

**Stalled jobs.** A job stuck in `active` (worker crashed, event loop blocked past the lock renewal) is detected by a periodic stalled-check and moved back to `wait` for reprocessing. In BullMQ v1 this — and delayed-job promotion — required a separate `QueueScheduler` instance; that class was removed and its responsibilities folded into `Worker`, so modern code no longer instantiates a scheduler[^3]. This is the single biggest source of stale tutorials online.

**Events.** `QueueEvents` consumes a Redis Stream (`XADD`/`XREAD`) that scripts append to on every transition, giving cross-process notifications. Local `Worker` events (`completed`, `failed`, `progress`) are in-process and cheaper; global events go through the stream.

**Flows.** `FlowProducer` adds a job tree where a parent does not become processable until its children complete (`waiting-children`), enabling fan-out/fan-in pipelines.

**Repeatable jobs / Job Schedulers.** Cron-like repeats were reworked into the "Job Scheduler" API in the v5 line, superseding the older `repeat` options; the scheduler produces delayed jobs on a schedule rather than storing a growing set of repeat keys[^4].

## Production Notes

- **`maxRetriesPerRequest` must be `null` on the connection.** BullMQ uses ioredis blocking commands (`BLMOVE`), and ioredis's default retry limit breaks them. If you pass an ioredis instance instead of options, you must set `maxRetriesPerRequest: null` (and typically `enableReadyCheck: false`). This is the most common first-run failure and BullMQ will warn about it.
- **Share Redis connections deliberately.** Every `Queue`, `Worker`, and `QueueEvents` opens connections; `Worker` and `QueueEvents` hold blocking connections that cannot be shared with non-blocking clients. On serverless/many-instance deploys this multiplies fast — budget Redis `maxclients` accordingly.
- **Completed/failed jobs accumulate forever by default.** Set `removeOnComplete` / `removeOnFail` (count or age) or your Redis memory grows unbounded. Retention is a per-job or per-queue default, not automatic.
- **Managed Redis quirks.** Some providers disable Lua scripting, `CLIENT` commands, or keyspace features BullMQ relies on; `maxmemory-policy` set to an eviction policy (e.g. `allkeys-lru`) can silently evict job keys and corrupt queues — it must be `noeviction`.
- **Redis Cluster.** Supported only if all of a queue's keys hash to the same slot; BullMQ uses hash-tag prefixes (`{queueName}`) for this, but cross-queue atomicity (e.g. flows spanning queues) constrains your cluster layout.
- **At-least-once, not exactly-once.** A worker that crashes after doing side effects but before acking will have its job re-run. Processors should be idempotent. There is no distributed transaction across your job and external systems.
- **Version skew across the fleet.** Because logic lives in Lua shipped with the client, running mixed BullMQ versions against one queue can mismatch scripts and key layouts. Upgrade producers and workers together; read the changelog for migrations between majors.

## When to Use / When Not

**Use when:**
- You already operate Redis and want durable background jobs without a new broker.
- You need retries/backoff, delays, priorities, rate limiting, cron repeats, or parent/child flows out of the box.
- Your workload is Node.js/TypeScript-first (best-supported surface).
- Throughput fits a single Redis instance's ceiling (tens of thousands of jobs/sec range, workload-dependent).

**Avoid when:**
- You need exactly-once semantics, long-running durable workflows with history, or complex saga orchestration — reach for a workflow engine.
- You don't want Redis as a hard dependency, or your ops team standardizes on a SQL/broker-backed queue.
- You need queue depth far beyond one Redis node's memory, or strong cross-queue transactional guarantees.
- Your stack is polyglot and depends on parity — the non-Node ports trail the Node.js feature set.

## Alternatives

- OptimalBits/bull — the predecessor; same author, still maintained for bugfixes but not new features. Use when you're on an existing Bull v3 codebase and don't need BullMQ additions.
- bee-queue/bee-queue — leaner Redis queue optimized for high-throughput short jobs; use when you want minimal features and maximum speed.
- graphile/worker — Postgres-backed job queue; use when Postgres is your only datastore and you want jobs in the same transactional boundary.
- temporalio/temporal — durable workflow engine with history and replay; use when you need long-running orchestration and exactly-once-ish semantics, not just a queue.
- agenda/agenda — MongoDB-backed scheduler; use when Mongo is your stack and cron-style scheduling is the primary need.

## History

| Version | Date | Notes |
|---------|------|-------|
| Bull 3.x | 2013 onward | Original Redis queue (OptimalBits); now maintenance-only[^1]. |
| BullMQ 1.0 | 2020 | TypeScript rewrite; atomic Lua scripts, `QueueEvents`, flows added later in the 1.x line. |
| 2.0 | 2022 | `QueueScheduler` removed — delayed/stalled handling moved into `Worker`[^3]. |
| 3.x / 4.x | 2022–2023 | Prioritized-jobs rework, deduplication, API refinements. |
| 5.0 | 2024 | Job Scheduler API supersedes legacy repeatable jobs[^4]. |
| Multi-language | 2024–2026 | Native Python, Rust, Elixir, and PHP ports added to the monorepo. |

## References

[^1]: Bull / BullMQ history and authorship — taskforce.sh. https://docs.bullmq.io/
[^2]: BullMQ Pro and Taskforce.sh dashboard. https://bullmq.io/#bullmq-pro
[^3]: BullMQ docs, "QueueScheduler" deprecation / migration to Worker. https://docs.bullmq.io/guide/jobs/stalled
[^4]: BullMQ docs, "Job Schedulers" (repeatable jobs successor). https://docs.bullmq.io/guide/job-schedulers

## Tags

nodejs, typescript, redis, job-queue, message-queue, background-jobs, task-queue, distributed-systems, lua, worker-pool
