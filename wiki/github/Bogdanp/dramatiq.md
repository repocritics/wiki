# Bogdanp/dramatiq

> A distributed task-processing library for Python 3 — Celery's design opinions, minus most of Celery's surface area.

[GitHub repo](https://github.com/Bogdanp/dramatiq) ·
[Official website](https://dramatiq.io) ·
[License: LGPL-3.0](https://github.com/Bogdanp/dramatiq/blob/master/COPYING)

## Overview

Dramatiq is a background/distributed task queue for Python, created by Bogdan
Popa and first published in 2017[^1]. It occupies the same niche as Celery —
enqueue a function call now, run it later in a pool of workers — but was
written as a deliberate reaction to Celery's size, configuration surface, and
default footguns. The pitch is a smaller, more opinionated library that makes
the reliable choices (automatic retries, message age limits, at-least-once
delivery) the defaults rather than the opt-ins.

The scope is intentionally narrow. Dramatiq supports exactly two brokers,
RabbitMQ and Redis, and treats them as first-class rather than plugging into a
generic transport abstraction[^2]. It has no bundled scheduler (no built-in
cron/beat), a deliberately minimal result backend, and no support for Windows.
That narrowness is the product: fewer knobs, fewer ways to configure yourself
into a corner. The tradeoff is real — teams that need SQS/Kafka transports,
complex canvas-style workflows, or a batteries-included periodic scheduler will
hit the edges of what Dramatiq chooses to do.

The library is LGPL-3.0 licensed, which is worth noting up front: it is
copyleft, unlike the BSD/MIT licensing of most of its competitors[^2]. For most
users importing it as a dependency this is unremarkable, but it is a
distinction legal review sometimes flags.

## Getting Started

```bash
pip install 'dramatiq[rabbitmq,watch]'   # RabbitMQ broker
# or
pip install 'dramatiq[redis,watch]'      # Redis broker
```

```python
# example.py
import dramatiq
import requests

@dramatiq.actor(max_retries=3)
def count_words(url):
    resp = requests.get(url)
    print(f"There are {len(resp.text.split())} words at {url!r}.")

if __name__ == "__main__":
    count_words.send("https://example.com")   # enqueue, returns immediately
```

```bash
dramatiq example        # start a pool of worker processes/threads
python example.py       # enqueue a message from anywhere
```

A function becomes a task by decorating it with `@dramatiq.actor`. Calling it
normally runs it inline; `.send()` (or `.send_with_options(...)`) enqueues it.
The `dramatiq` CLI boots the workers.

## Architecture / How It Works

The core abstraction is the **actor** — a plain function wrapped by
`@dramatiq.actor`. Enqueuing produces a **message** (actor name + args +
options) that is serialized (JSON by default) and pushed to a **broker**. A
**worker** process pulls messages and dispatches them across a pool of OS
threads.

**Concurrency model.** `dramatiq` launches multiple worker *processes*, each
running multiple *threads* (`--processes` / `--threads`). Because of the GIL,
CPU-bound actors do not parallelize across threads within a process — you scale
those with more processes. Thread-based concurrency is a good fit for the
I/O-bound work that dominates task queues, and it keeps memory lower than
process-per-worker designs. Dramatiq is not asyncio-native; async actors exist
but the runtime is thread-first.

**Middleware pipeline.** Almost every feature is a middleware wrapping message
processing: `Retries`, `TimeLimit`, `ShutdownNotifications`, `AgeLimit`,
`Callbacks`, `Pipelines`, `GroupCallbacks`, `CurrentMessage`, and `Prometheus`.
This is the main extension point — rate limiters, custom serialization, and
metrics all hook in here.

**RabbitMQ broker.** Uses AMQP acknowledgements. For each queue, Dramatiq
provisions companion queues: a delay queue (`.DQ`) for `delay=`/retry
scheduling and a dead-letter queue (`.XQ`) for messages that exhaust retries or
exceed their age limit[^3]. Delivery is at-least-once: a message is only acked
after the actor returns successfully.

**Redis broker.** Implemented with server-side Lua scripts and a heartbeat/ack
scheme. If a worker dies mid-task, its unacknowledged messages are requeued
once its heartbeat lapses[^4]. This gives at-least-once semantics on a
transport that has no native ack, at the cost of Redis being a full message
store (and its memory being the ceiling on backlog).

**Retries by default.** Unhandled exceptions schedule a retry with exponential
backoff. This is on unless you set `max_retries=0` — a philosophical inversion
of Celery, where retries are opt-in per call.

## Production Notes

The following are the caveats operators actually hit:

- **Idempotency is mandatory.** At-least-once delivery means an actor can run
  more than once for a single logical message (worker crash after side effect,
  before ack; requeue after heartbeat lapse). Every actor must tolerate being
  replayed. This is inherent to the delivery guarantee, not a bug.
- **Retry storms.** Because retries are on by default with a generous default
  ceiling, a systemic downstream failure (a database being down) turns into a
  self-inflicted backlog of exponentially-backed-off retries. Set sane
  `max_retries`, `min_backoff`/`max_backoff`, and consider a `retry_when`
  predicate for non-retryable errors.
- **Time limits are cooperative-ish.** `TimeLimit` raises an exception *in the
  worker thread*, but Python can only deliver it at bytecode boundaries — a
  blocked C extension or a syscall (e.g. a socket with no timeout) will not be
  interrupted until it returns. Always set client-side timeouts on I/O; don't
  rely on `time_limit` to unstick a hung `requests` call.
- **Prefetch and long tasks.** Workers prefetch messages proportional to their
  thread count. A pool of long-running actors can prefetch-hoard a queue and
  starve other workers of short jobs. Tune prefetch (RabbitMQ) or split slow
  and fast work into separate queues/deployments.
- **No built-in scheduler.** There is no `celery beat` equivalent in the core.
  Periodic/cron work needs `periodiq` (by the same author) or APScheduler
  wired to `.send()`. Plan for this before you need it[^5].
- **Results backend is deliberately thin.** Storing large results in Redis is
  an antipattern the docs steer you away from; treat actors as fire-and-forget
  and persist real output to your own store.
- **Prometheus exposition.** The `Prometheus` middleware exposes metrics over
  an HTTP port per worker; in containerized/multi-process setups you must plan
  the port and scrape topology (it writes to a shared multiprocess dir).
- **No Windows.** The worker relies on `fork`; production and development are
  POSIX-only.

## When to Use / When Not

**Use when:**
- You run RabbitMQ or Redis already and want a task queue that is reliable by
  default without deep configuration.
- You value a small, legible codebase you can read end-to-end over a maximal
  feature set.
- Your workload is I/O-bound (HTTP, DB, external APIs) — the thread model fits.
- You want retries, dead-lettering, and age limits without wiring them up.

**Avoid when:**
- You need a transport other than RabbitMQ/Redis (SQS, Kafka, a database as
  broker).
- You need a rich built-in scheduler or complex workflow canvas (chords, chains
  with elaborate graph semantics) as a first-class feature.
- Your stack is asyncio-first and you want native `async def` task execution.
- LGPL copyleft is a problem for your distribution model, or you deploy on
  Windows.

## Alternatives

- celery/celery — the incumbent; use it when you need many broker/result
  backends, a built-in scheduler, and canvas workflows, and can absorb the
  configuration and operational complexity.
- rq/rq — Redis-only, fork-per-job, no threads; use it for simpler workloads
  where you want minimal moving parts and don't need Dramatiq's middleware.
- coleifer/huey — small queue with a built-in periodic scheduler and
  Redis/SQLite backends; use it when you want cron-style tasks bundled in.
- procrastinate-org/procrastinate — PostgreSQL as the broker; use it when you
  want transactional enqueue (task and business write in one DB transaction)
  and no extra infrastructure.
- taskiq-python/taskiq — async-first distributed task queue; use it when your
  codebase is asyncio-native and you want `async def` tasks as the primary
  model.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017 | Initial public release on PyPI; RabbitMQ + Redis brokers[^1]. |
| 1.0 | 2018 | First stable release; middleware pipeline and API settled. |
| 1.x | 2019–2026 | Ongoing 1.x line; Python 3 only, POSIX only, LGPL-3.0[^2]. |

(Point-release dates are intentionally omitted where not verified; see the
project changelog for exact version history[^6].)

## References

[^1]: dramatiq on PyPI — release history. https://pypi.org/project/dramatiq/#history
[^2]: dramatiq documentation — "Motivation" and installation. https://dramatiq.io/
[^3]: dramatiq documentation — Message delays, dead letters, and the `.DQ`/`.XQ` queues. https://dramatiq.io/guide.html
[^4]: dramatiq API reference — Redis broker. https://dramatiq.io/reference.html
[^5]: periodiq — periodic task scheduler for Dramatiq (same author). https://gitlab.com/bersace/periodiq
[^6]: dramatiq changelog. https://dramatiq.io/changelog.html

## Tags

python, task-queue, background-jobs, distributed, rabbitmq, redis, actor-model, message-queue, worker, at-least-once
