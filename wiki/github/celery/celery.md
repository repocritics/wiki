# celery/celery

> Distributed task queue for Python — the default answer to "run this function outside the request/response cycle."

[GitHub repo](https://github.com/celery/celery) ·
[Official website](https://docs.celeryq.dev) ·
License: BSD-3-Clause[^1]

## Overview

Celery is a Python framework for running work asynchronously across processes and machines. A client (usually a web app) enqueues a message describing a unit of work; long-lived worker processes pull messages off a broker and execute the corresponding Python function. It has been the incumbent solution for background jobs in the Python ecosystem since roughly 2010 — the thing Django and Flask apps reach for when a request needs to send email, process an upload, or fan out to an external API without blocking[^2].

The defining architectural choice is that Celery is *not* itself a queue. It is a producer/consumer layer that sits on top of a separate message broker (RabbitMQ, Redis, Amazon SQS, or Google Pub/Sub) and, optionally, a separate result backend (Redis, a database, S3, and others). This decoupling is the source of both its flexibility and most of its operational pain: correctness and delivery semantics depend heavily on which broker you chose and how you configured it, and the failure modes differ enough that "Celery works on my machine" is a common false confidence.

Celery is broad rather than minimal. Beyond fire-and-forget tasks it ships a workflow layer ("Canvas" — chains, groups, chords, maps), a cron-like scheduler (Celery Beat), retry/rate-limit/time-limit machinery, and a large matrix of concurrency pools and serializers. That surface area is why it fits nearly every job-queue need and also why smaller, more opinionated competitors keep appearing.

## Getting Started

```bash
pip install "celery[redis]"   # broker extra; use celery[amqp] for RabbitMQ, celery[sqs] for SQS
```

```python
# tasks.py
from celery import Celery

app = Celery("tasks", broker="redis://localhost:6379/0",
             backend="redis://localhost:6379/1")

@app.task
def add(x, y):
    return x + y
```

```bash
# Run a worker (prefork pool)
celery -A tasks worker --loglevel=info
```

```python
# Enqueue from anywhere; .delay() is shorthand for .apply_async()
>>> from tasks import add
>>> result = add.delay(4, 6)
>>> result.get(timeout=10)   # blocks; requires a result backend
10
```

## Architecture / How It Works

Celery is a thin orchestration layer over a family of sibling libraries maintained by the same project: **kombu** (broker abstraction / messaging), **billiard** (a fork of `multiprocessing` with fixes for worker pools), **amqp** (the RabbitMQ protocol client), and **vine** (promises). Understanding Celery mostly means understanding kombu's transport model, because that is where broker-specific behavior lives.

The message flow: a task call serializes arguments (JSON by default since 4.0; pickle/yaml/msgpack are opt-in) and publishes a message to an **exchange/queue** on the broker. A worker's main process consumes messages and dispatches them to a **pool** of executors.

Concurrency pools are pluggable and their tradeoffs matter:
- **prefork** (default) — OS processes via billiard. Best for CPU-bound work and for isolating crashes; each child holds its own memory.
- **eventlet** / **gevent** — greenlet-based cooperative concurrency for I/O-bound tasks; thousands of concurrent tasks per worker, but any C extension that blocks the event loop stalls all of them.
- **solo** — single-threaded, in-process; useful for debugging and for tasks that must not run concurrently.
- **threads** — a thread pool; simpler than gevent but subject to the GIL.

**Celery Beat** is a separate scheduler process that periodically publishes messages for cron-style and interval tasks. It is a single point of coordination: running two Beat instances double-fires schedules unless you add a lock (`celery-redbeat` and similar).

**Canvas** composes tasks into workflows. A `chain` runs tasks in sequence passing results forward; a `group` runs them in parallel; a `chord` runs a group and then a callback once all members finish. Chords in particular require the result backend to track completion, which couples workflow correctness to backend reliability.

The result backend is optional and frequently misunderstood: it exists only to store return values and state. Many production systems run *without* one (`ignore_result=True`), because storing every result creates load and, on Redis, unbounded key growth.

## Production Notes

Celery is one of those tools that is trivial to start and full of edges at scale. The most common footguns:

- **Redis is not RabbitMQ.** Redis as a broker has no native message acknowledgement; kombu emulates it with a **visibility timeout**. If a task runs longer than the timeout (default 1 hour), the message becomes visible again and *another worker runs it a second time*. Long tasks on Redis must raise `broker_transport_options={'visibility_timeout': ...}` — a duplicate-execution bug that only appears under real load.
- **Acknowledgement timing.** By default a task is acked when the worker *receives* it, so a worker crash mid-task loses the message. `task_acks_late=True` acks on completion (at-least-once semantics), but then your tasks must be idempotent, because retries and visibility timeouts will re-run them.
- **Prefetch multiplier.** Workers prefetch `worker_prefetch_multiplier × concurrency` messages. The default (4) is wrong for a mix of long and short tasks — a few slow tasks hog prefetched messages while other workers idle. Long-task workers typically set it to 1.
- **Memory leaks / bloat.** The prefork pool does not release memory between tasks; a leaky dependency grows child RSS indefinitely. The standard mitigation is `worker_max_tasks_per_child` (recycle a child after N tasks) or `worker_max_memory_per_child`.
- **Result backend pressure.** Using Redis as a result backend under high throughput produces large numbers of short-lived keys; set `result_expires` and consider disabling results for tasks that don't need them.
- **Monitoring.** Celery's own introspection (`celery inspect`, events) is limited; most shops run **Flower** for a live dashboard, or scrape Prometheus via a community exporter. Silent task loss is hard to detect without external observability.
- **Windows is unsupported.** The maintainers explicitly do not support Microsoft Windows and ask that related issues not be filed[^3].

## When to Use / When Not

**Use when:**
- You have a Python web app and need background jobs, scheduled tasks, retries, and fan-out with one well-trodden tool.
- You already run RabbitMQ or Redis and want at-least-once delivery with mature broker support.
- You need workflow composition (chains/groups/chords) rather than isolated jobs.

**Avoid when:**
- Your needs are simple and Redis-only — a lighter library (RQ, Dramatiq, Huey) has fewer moving parts and clearer defaults.
- You need durable, long-running, resumable workflows with strong exactly-once guarantees — a purpose-built durable-execution engine fits better than Canvas + a result backend.
- Your stack is async-native (`asyncio`) — Celery's task model predates and sits awkwardly beside asyncio; an asyncio-first queue is a more natural match.
- You want minimal operational surface — Celery adds a broker, optionally a result backend, worker fleets, and a Beat process to run and monitor.

## Alternatives

- rq (python-rq/rq) — use instead when you're Redis-only and want a small, readable codebase over Celery's feature breadth.
- Bogdanp/dramatiq — use instead when you want Celery-style features with saner defaults and simpler configuration.
- coleifer/huey — use instead for lightweight scheduling and tasks with minimal dependencies.
- python-arq/arq — use instead when your app is asyncio-first and you want native `async def` tasks.
- temporalio/temporal — use instead when you need durable, resumable, exactly-once workflows rather than best-effort task dispatch.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2009-04 | Initial release by Ask Solem[^2]. |
| 1.0 | 2010-02 | First stable line; RabbitMQ-centric. |
| 3.0 | 2012 | Canvas workflow primitives (chains/groups/chords). |
| 4.0 | 2016-11 | New message protocol, JSON default serializer, kombu 4[^4]. |
| 5.0 | 2020-09 | Python 3.6+ only; dropped Python 2 support[^5]. |
| 5.3 | 2023 | Redis/SQS improvements, Python 3.11 support. |
| 5.6.2 | 2026 | Current release line; supports Python 3.9–3.13, last line to support 3.9[^3]. |

## References

[^1]: The project README states the software is licensed under the New BSD License (BSD-3-Clause); GitHub's license detector reports `NOASSERTION` because the `LICENSE` file is not an exact SPDX match. https://github.com/celery/celery/blob/main/LICENSE
[^2]: Celery documentation, "Introduction to Celery" / project history. https://docs.celeryq.dev/en/stable/getting-started/introduction.html
[^3]: Celery README, "What do I need?" — supported Python versions and Windows non-support (v5.6.x). https://github.com/celery/celery/blob/main/README.rst
[^4]: Celery 4.0 "latentcall" release notes. https://docs.celeryq.dev/en/stable/history/whatsnew-4.0.html
[^5]: Celery 5.0 release notes — Python 2 removal. https://docs.celeryq.dev/en/stable/history/whatsnew-5.0.html

## Tags

python, task-queue, distributed-systems, background-jobs, message-broker, rabbitmq, redis, celery, async, job-scheduler, workflow
