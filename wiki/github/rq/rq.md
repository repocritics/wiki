# rq/rq

> Redis-backed job queue for Python — deliberately small, fork-per-job by default, and honest about its `pickle` footgun.

[GitHub repo](https://github.com/rq/rq) ·
[Official website](https://python-rq.org) ·
License: BSD (repo `LICENSE`; GitHub reports no SPDX match)

## Overview

RQ (Redis Queue) is a Python library for enqueueing function calls and running
them in background worker processes, using Redis or Valkey as the only backing
store. It was created around 2011 by Vincent Driessen as a lightweight
alternative to Celery and the AMQP-based queueing stacks of the era[^1], trading
broker flexibility and feature breadth for a small API you can read in an
afternoon. As of 2026 it has ~10.7k stars, ~1.5k forks, and is still actively
maintained (last push mid-2026)[^2]. Maintenance is now led by Stamps, an
Indonesian CRM/order-management company that funds ongoing work[^3].

The design philosophy is "no broker, no daemon, no DSL." You define an ordinary
function, `queue.enqueue(fn, arg)` serializes the call into Redis, and a
separate `rq worker` process pops it and runs it. There is no separate scheduler
service required for basic use, no message-broker abstraction layer, and no
result backend to configure — Redis is all three. The cost of that simplicity is
that RQ inherits Redis's durability model (in-memory, optionally persisted) and
that its default serializer is `pickle`, which makes an untrusted or
network-exposed Redis a remote-code-execution surface (see Production Notes).

RQ is synchronous by design: jobs are plain blocking functions run in OS
processes, not coroutines. Teams that need `asyncio`-native execution or a
non-Redis broker consistently reach for a different library rather than bending
RQ to fit.

## Getting Started

```console
$ pip install rq
$ redis-server            # RQ needs Redis >= 5 or Valkey >= 7.2
```

```python
# tasks.py
import requests

def count_words_at_url(url):
    resp = requests.get(url)
    return len(resp.text.split())
```

```python
# enqueue.py
from redis import Redis
from rq import Queue
from tasks import count_words_at_url

queue = Queue(connection=Redis())
job = queue.enqueue(count_words_at_url, "https://python-rq.org")
print(job.id)          # poll job.result later; None until finished
```

```console
$ rq worker                # start a worker to drain the "default" queue
```

## Architecture / How It Works

`Queue.enqueue` builds a `Job` — a dict of the target function's import path,
serialized args/kwargs, timeout, and metadata — and `LPUSH`es its ID onto a
Redis list (`rq:queue:<name>`), storing the job body under a separate hash. A
worker does a blocking `BRPOP` across the queue lists it was started with, in
argument order, which is how priority works: `rq worker high low` fully drains
`high` before touching `low`. There is no fair scheduling across queues — order
is strict precedence. Single-queue priority is coarser: `enqueue(..., at_front=True)`
`RPUSH`es to the head instead of the tail.

The default `Worker` forks a child process per job. The child runs the function;
the parent monitors it, enforces the hard timeout via `SIGALRM`/kill, and reaps
the result. This gives crash, memory-leak, and timeout isolation at the price of
a `fork()` per job. Two alternatives trade that off: `SimpleWorker` runs jobs
in-process (no fork — roughly 6x faster in a hello-world microbenchmark, but no
isolation and no hard timeout on POSIX)[^4], and `SpawnWorker` uses `spawn`
instead of `fork` for platforms/config where fork is unsafe. `worker-pool -n N`
runs N worker processes under one command for simple local scale-out.

Serialization defaults to `pickle`, which is why arbitrary Python objects can be
passed as arguments and returned as results. Swapping in `JSONSerializer`
restores safety but restricts arguments to JSON primitives. Job state (queued,
started, deferred, finished, failed, canceled) lives in Redis registries; the
`StartedJobRegistry` plus per-worker heartbeats are how RQ detects a worker that
died mid-job and moves the job to `FailedJobRegistry`.

Scheduling has been folded into core: `enqueue_at` / `enqueue_in` put jobs on a
sorted set that a worker started with `--with-scheduler` (or a dedicated
scheduler) moves onto the queue when due. Newer releases add a built-in cron
runner (`rq cron config.py`, RQ >= 2.5) supporting both fixed intervals and
standard cron syntax, `Repeat` for N re-runs, `Retry` with per-attempt
intervals, `unique=True` dedup, `RateLimit` concurrency limits (added in
2.11.0), and outbound `Webhook`s on job finish/fail. These are convenience
layers over the same list-and-registry primitives, not a new engine.

## Production Notes

- **`pickle` is the headline footgun.** The default serializer will execute
  arbitrary code on unpickle, so any process that can write to your Redis can run
  code inside your workers. Only point RQ at a Redis you fully trust, and prefer
  `JSONSerializer` when your job arguments allow it. This is called out in RQ's
  own README, not just by critics[^5].
- **Durability is Redis's durability.** A job is one Redis list entry. If Redis
  is not configured with AOF/RDB persistence, a crash loses queued jobs; if it is
  memory-pressured and evicting keys, jobs and results can silently disappear. Run
  the Redis backing RQ with `maxmemory-policy noeviction` and appropriate
  persistence.
- **Fork cost is usually irrelevant — until it isn't.** The default `Worker`'s
  per-job fork overhead is negligible next to real work (HTTP calls, image
  processing). Only switch to `SimpleWorker` for sub-millisecond jobs at high
  throughput, and only when the code is trusted and leak-free, because you also
  give up crash isolation and hard timeouts.
- **No native async.** Jobs run in processes, not an event loop. Enqueueing from
  async web code is fine (the enqueue is a quick Redis write), but if your *jobs*
  are I/O-bound coroutines you will be wrapping event loops per job or picking a
  different library.
- **Run workers under a process supervisor.** RQ ships no daemon; production
  setups use systemd, supervisor, or a container orchestrator to keep
  `rq worker` / `worker-pool` alive and restart on exit. Horizontal scale is
  "start more worker processes."
- **Monitoring is BYO.** Core RQ has a CLI (`rq info`) but no web UI; teams
  bolt on `rq-dashboard` or `rqmonitor`. Failed jobs accumulate in the
  `FailedJobRegistry` and need an explicit retention/requeue policy or they grow
  unbounded.

## When to Use / When Not

**Use when:**
- You already run Redis/Valkey and want background jobs with minimal moving parts.
- Your jobs are ordinary synchronous Python (emails, reports, image/media work).
- You value a small, readable codebase and a fast path to first working worker.
- Coarse queue-precedence priority and simple scheduling cover your needs.

**Avoid when:**
- You need a broker other than Redis, or Redis-grade durability isn't enough.
- Your jobs are `asyncio`-native and you want in-loop concurrency.
- You require rich routing, workflows/canvas (chords, chains, groups), or
  multi-broker federation — that's Celery's territory.
- You can't trust every writer to your Redis and can't move off `pickle`.

## Alternatives

- celery/celery — reach for it when you need multiple brokers (AMQP/Redis),
  complex workflows, or a mature ecosystem, and accept the heavier operational
  footprint.
- coleifer/huey — use when you want an RQ-sized library with optional
  SQLite/in-memory backends and built-in periodic tasks.
- Bogdanp/dramatiq — use when you want a middleware-based actor model with
  first-class RabbitMQ support and stronger delivery guarantees than a Redis list.
- samuelcolvin/arq — use when your jobs are `asyncio` coroutines and you want a
  Redis queue designed around the event loop instead of `fork()`.
- procrastinate-org/procrastinate — use when you'd rather back the queue with
  PostgreSQL (transactional enqueue, no separate Redis) than Redis.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-11-14 | First public commit; created as a lighter alternative to Celery/Resque[^1]. |
| 1.0 | 2019-07 | First stable major; API stabilization after years of 0.x. |
| 2.0 | 2024-11 | Major line; scheduler and worker improvements consolidated into core. |
| 2.5 | 2025 | Built-in `rq cron` interval + cron-syntax scheduling. |
| 2.11.0 | 2025 | `RateLimit` concurrency limits for jobs sharing a key. |

## References

[^1]: RQ README, "Project history" — inspired by Celery, Resque, and a Flask snippet; built as a lightweight alternative to AMQP-based queueing. https://github.com/rq/rq#project-history
[^2]: GitHub API, repos/rq/rq — 10,665 stars, 1,478 forks, last push 2026-07-12 (fetched 2026-07). https://github.com/rq/rq
[^3]: RQ README — "RQ is maintained by Stamps, an Indonesian based company." https://github.com/rq/rq
[^4]: RQ README, "Notes on Performance" — SimpleWorker ~6x faster in a hello-world microbenchmark by skipping fork/spawn. https://github.com/rq/rq/blob/master/docs/benchmark.md
[^5]: RQ README, "Security" — pickle default serializer is not secure; use JSONSerializer for untrusted input. https://python-rq.org/docs/jobs/
[^6]: RQ documentation. https://python-rq.org

## Tags

python, job-queue, task-queue, redis, valkey, background-jobs, workers, distributed, scheduling, async
