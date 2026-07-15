# agronholm/apscheduler

> In-process task scheduler for Python — cron/interval/date triggers with pluggable persistence, not a distributed task queue (until v4).

[GitHub repo](https://github.com/agronholm/apscheduler) ·
[Official website](https://apscheduler.readthedocs.io/) ·
[License: MIT](https://github.com/agronholm/apscheduler/blob/master/LICENSE.txt)

## Overview

APScheduler ("Advanced Python Scheduler") schedules Python callables to run
later — once at a date, on an interval, or on a cron-style calendar — and
optionally persists those schedules so they survive process restarts. It is
maintained primarily by Alex Grönholm (also author of anyio and typeguard) and
has been a default answer to "how do I run a function every N minutes inside my
Python app" since the 3.x rewrite in 2014[^1]. At ~7.6k stars it is one of the
most-depended-on scheduling libraries on PyPI, pulled in by Django/Flask apps,
data pipelines, and home-automation projects alike.

The defining tension is scope. Through the entire 3.x line — still the version
almost everyone runs in production — APScheduler is fundamentally an
**in-process, single-node** scheduler. It has no leader election, no locking,
and no work distribution. The persistent job stores make it *look* like
infrastructure you can scale horizontally, but running two scheduler processes
against the same store does not coordinate them — it double-fires jobs. That gap
is exactly what the long-running 4.0 rewrite exists to close, by splitting the
scheduler from the worker and adding shared data stores plus event brokers for
true multi-node operation[^2].

As of this writing 4.0 is still shipped as a **pre-release** and the README
warns in bold not to use it in production, with no promised migration path[^2].
So the practical library is 3.x, and most of the caveats below are about not
mistaking it for something bigger than it is.

## Getting Started

```bash
pip install apscheduler          # installs the stable 3.x line
# pip install "apscheduler>=4.0a1"  # opt-in to the 4.0 pre-release only
```

```python
# 3.x — the version you almost certainly want
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.cron import CronTrigger

def report():
    print("running the 09:00 report")

scheduler = BackgroundScheduler(timezone="UTC")
scheduler.add_job(report, CronTrigger(hour=9, minute=0), id="daily_report",
                  replace_existing=True)
scheduler.start()   # non-blocking; runs jobs on a background thread pool
```

Use `BlockingScheduler` for a script whose only job is to schedule, or
`AsyncIOScheduler` inside an asyncio event loop.

## Architecture / How It Works

3.x composes four replaceable pieces, and understanding the split explains most
of its behavior:

1. **Schedulers** — the loop that decides what runs when.
   `BackgroundScheduler` (own thread), `BlockingScheduler`, `AsyncIOScheduler`,
   plus Gevent/Tornado/Twisted/Qt variants. You pick one to match your app's
   concurrency model.
2. **Job stores** — where scheduled jobs live. `MemoryJobStore` (default, lost
   on restart) or persistent backends: `SQLAlchemyJobStore` (any SQL DB),
   `MongoDBJobStore`, `RedisJobStore`, `ZooKeeperJobStore`. Persistence
   serializes the job's callable reference and args via pickle.
3. **Executors** — how a triggered job actually runs. `ThreadPoolExecutor`
   (default) or `ProcessPoolExecutor` for CPU-bound work.
4. **Triggers** — `date`, `interval`, `cron`, and combining triggers (`and`/
   `or`) for expressing "every weekday at 9 but not on holidays" style rules.

The scheduler wakes, asks each job store for jobs due before now, and hands them
to an executor. Three per-job knobs govern edge behavior and are the source of
most surprises: **`misfire_grace_time`** (how late a run may start before it is
skipped), **`coalesce`** (collapse multiple missed runs into one), and
**`max_instances`** (cap concurrent runs of the same job). If your process was
asleep or overloaded, these decide whether missed runs fire, get dropped, or
pile up.

Crucially, none of these components coordinate across processes. The job store
is shared *state*, not a shared *work queue*. 4.0 reworks this entirely:
scheduler and worker become separate roles, a **data store** (PostgreSQL, MySQL,
SQLite, MongoDB) holds schedules and jobs, and an **event broker** (PostgreSQL,
Redis, MQTT) lets multiple nodes react to changes — giving the horizontal
scaling and degree of high availability 3.x never had[^2]. 4.0 also unifies the
sync and async APIs behind a single scheduler class.

## Production Notes

- **The multi-instance footgun (3.x).** Two schedulers pointed at one
  `SQLAlchemyJobStore` will each independently see a due job and run it — there
  is no locking or leader election. If you deploy N replicas of a web app that
  each start a `BackgroundScheduler`, every cron job fires N times. Mitigations:
  run the scheduler in exactly one dedicated process, or gate startup behind an
  external lock/leader election. This is the single most common APScheduler
  incident.
- **Pickled callables.** Persistent job stores pickle a reference to your
  function (by import path) and its arguments. Renaming or moving the function,
  or passing an unpicklable argument (a DB session, an open socket), breaks
  restoration on restart. Keep jobs as top-level named functions taking simple
  args, and look those args up inside the job.
- **Timezones.** 3.x historically used pytz and is strict about tz-aware
  scheduling; ambiguous/naive datetimes around DST are a recurring bug source.
  Set an explicit `timezone=` on the scheduler rather than relying on the
  server's local zone.
- **Missed runs after downtime.** Default `misfire_grace_time` is small, so jobs
  due while the process was down are silently skipped unless you raise it and/or
  set `coalesce=True`. Long-outage catch-up is not automatic.
- **Thread pool starvation.** The default executor is a bounded thread pool;
  slow jobs plus low `max_instances` headroom cause queuing and misfires. Move
  blocking/CPU work to `ProcessPoolExecutor` or an external queue.
- **Don't reach for 4.0 yet.** Its API can change without a backward-compatible
  migration path per the maintainer's own warning[^2]; pin to a specific alpha
  and expect churn if you experiment.

## When to Use / When Not

**Use when:**
- You need cron/interval/one-off scheduling *inside* a single Python process.
- You want persistence so schedules survive restarts, in one authoritative
  scheduler process.
- You need flexible trigger logic (combining triggers, custom trigger classes)
  richer than a plain crontab.

**Avoid when:**
- You need distributed execution across many workers with at-least/at-most-once
  guarantees — that is a task-queue problem, not a scheduler one (until 4.0
  matures).
- You are running multiple app replicas and can't isolate a single scheduler
  process — you will double-fire.
- All you need is "run this shell script at 3am on a box" — system cron or
  systemd timers are simpler and need no process kept alive.

## Alternatives

- celery/celery — distributed task queue with a beat scheduler; use it when jobs
  must run across many worker nodes with retries and result backends.
- dbader/schedule — tiny human-readable in-process scheduler; use it when you
  want periodic jobs and zero persistence or trigger complexity.
- rq/rq (with rq-scheduler) — Redis-backed job queue; use it when you already
  run Redis and want simple background jobs plus scheduling.
- Bogdanp/dramatiq — distributed task processing; use it when you want Celery's
  distribution with a smaller, more opinionated API.
- samuelcolvin/arq — asyncio + Redis job queue; use it when your stack is async
  and you want cron-like jobs on Redis.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | ~2009 | Original single-module scheduler. |
| 2.x | ~2011 | Persistent job store support introduced. |
| 3.0 | 2014 | Full rewrite: pluggable scheduler/jobstore/executor/trigger design[^1]. |
| 3.x | 2014–present | The stable, widely deployed line; ongoing maintenance releases. |
| 4.0 | pre-release | Sync+async unification, scheduler/worker split, shared data stores + event brokers for multi-node; still alpha, not for production[^2]. |

## References

[^1]: APScheduler documentation — user guide and API reference (3.x design: schedulers, job stores, executors, triggers). https://apscheduler.readthedocs.io/en/3.x/
[^2]: APScheduler README / 4.0 documentation — pre-release warning and data-store + event-broker architecture. https://github.com/agronholm/apscheduler and https://apscheduler.readthedocs.io/en/master/

## Tags

python, scheduling, cron, task-queue, background-jobs, asyncio, job-scheduler, automation, library, persistence
