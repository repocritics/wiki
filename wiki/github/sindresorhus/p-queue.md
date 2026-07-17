# sindresorhus/p-queue

> In-process promise queue with concurrency control and interval-based rate limiting.

[GitHub repo](https://github.com/sindresorhus/p-queue) ·
[License: MIT](https://github.com/sindresorhus/p-queue/blob/main/license)

## Overview

p-queue is a single-process, in-memory queue for async (or sync) functions that
caps how many run at once and, optionally, how many may start per unit of time.
It is one of the most-used members of Sindre Sorhus's `p-*` promise-utility
family, sitting a rung above `p-limit`: where `p-limit` only bounds concurrency,
p-queue adds priority ordering, per-task timeouts, `AbortSignal` cancellation,
pause/resume, interval rate limiting, and an event stream for progress
tracking. The typical use is throttling calls to a REST API or spacing out
CPU/memory-heavy work so it doesn't all fire simultaneously.

The defining constraint is the word *in-process*. p-queue holds tasks as
closures in a JS array; there is no persistence, no cross-process coordination,
and no recovery. If the Node process restarts, everything queued is gone. The
README states this explicitly, pointing servers at a Redis-backed job queue
instead.[^1] Read that as the boundary: p-queue is for orchestrating work
*within* one event loop, not for durable background jobs.

As of 2026 the project is declared **feature complete** by its author — pull
requests are reviewed but no new development is planned, and there is no email
support.[^1] For a utility this small and stable, "done" is a reasonable state
rather than abandonment, but it means new capabilities (distributed limiting,
persistence) will not arrive here.

## Getting Started

```sh
npm install p-queue
```

**Native ESM only.** The package dropped its CommonJS export; a `require()` from
CJS will fail. Projects still on CommonJS must convert to ESM or use a dynamic
`import()`.[^2]

```js
import PQueue from 'p-queue';

const queue = new PQueue({concurrency: 2});

const urls = ['https://a.example', 'https://b.example', 'https://c.example'];

await Promise.all(
	urls.map(url =>
		queue.add(async () => {
			const res = await fetch(url);
			return res.json();
		})
	)
);
// At most 2 fetches run at any moment; the third waits for a free slot.
```

## Architecture / How It Works

`PQueue` is a subclass of `EventEmitter3`.[^3] Internally it keeps a pluggable
priority queue (the default `queueClass` is a small array-backed structure
sorted by descending `priority`, ties broken by insertion order) plus a counter
of currently `pending` tasks. Scheduling is a loop: whenever a task settles or a
new one is added, `add()` checks whether a slot is free under both the
concurrency limit and the interval budget, and if so dequeues the
highest-priority task and runs it.

Two independent throttles compose:

- **Concurrency** (`concurrency`) — the max number of tasks executing
  simultaneously. This is the `p-limit` behavior.
- **Rate limiting** (`intervalCap` + `interval`) — the max number of tasks that
  may *start* within each rolling `interval` window. Default mode resets the
  count at fixed interval boundaries, which permits bursts across a boundary (2
  at 999 ms, 2 more at 1000 ms). Setting `strict: true` switches to a
  sliding-window algorithm that tracks individual start timestamps and forbids
  more than `intervalCap` starts in *any* `interval`-length window — more
  predictable, and more expensive because it retains per-task timestamps.

A task is a closure passed to `.add(fn, options)`, which returns a promise that
settles when the task *finishes*, not when it is enqueued. The `fn` receives
`{signal}` so a task can observe an `AbortSignal`; aborting a still-queued task
removes it and rejects its `add()` promise, but aborting a *running* task only
signals — the task's own code must honor it. Priorities can be mutated after
enqueue via `.setPriority(id, priority)`, which only has effect when a finite
concurrency limit is set.

Beyond `add`/`addAll`, the surface is largely observational: `.size`,
`.pending`, `.isPaused`, `.isRateLimited`, `.isSaturated`, `.runningTasks`, and
a matched set of one-shot promises (`onEmpty`, `onIdle`, `onPendingZero`,
`onSizeLessThan`, `onRateLimit`, `onError`) alongside repeating events (`active`,
`completed`, `error`, `empty`, `idle`, `add`, `next`, `rateLimit`). The
`empty` vs `idle` distinction trips people up: `empty` means nothing is *waiting*
(the queue array is drained) while `idle` additionally requires `pending === 0`
(nothing still *running*). Drain with `onIdle`, not `onEmpty`.

## Production Notes

- **`.clear()` orphans promises.** Clearing the queue does *not* settle the
  `add()` promises of the tasks it removed — they never resolve or reject. Any
  code `await`ing them hangs. To drop queued work while still settling its
  promises, cancel with an `AbortSignal` instead of calling `.clear()`.[^4]
- **`await queue.add(...)` can defeat the point.** Awaiting a single `add()`
  serializes the caller against that one task. To fan out N tasks under the
  concurrency cap, collect the promises and `await Promise.all(...)`, or use
  `onIdle()`.
- **Unhandled rejections.** A rejecting task rejects its `add()` promise *and*
  emits `error`. If you rely on the event and never `.catch()` the promise (a
  common pattern with fire-and-forget `add()`), Node may report an unhandled
  rejection and, on some configurations, exit. Handle the promise even when you
  listen for `error`.
- **Backpressure is manual.** p-queue happily accepts an unbounded backlog; a
  producer faster than the consumer grows the in-memory array without limit.
  Gate producers with `await queue.onSizeLessThan(n)` or the newer
  `onRateLimit` / `isSaturated` signals before adding more.
- **Timeouts start at dequeue.** A per-task `timeout` counts from when the task
  begins executing, not from enqueue; time spent waiting behind the concurrency
  or interval limit is not counted. A timed-out task throws `TimeoutError` but,
  like abort, cannot forcibly stop already-running work.
- **No durability, no clustering.** Restart loses the queue; multiple processes
  do not share limits. For rate limits that must hold across replicas you need
  an external coordinator (Redis) — p-queue cannot provide it.

## When to Use / When Not

**Use when:**
- You need to cap concurrency and/or start-rate for async work inside one Node
  process (API clients, scrapers, batch fetch/upload, migration scripts).
- You want priority ordering, pause/resume, or progress events on top of a
  plain concurrency limit.
- You are in a browser or a short-lived process where an external queue would be
  overkill.

**Avoid when:**
- Jobs must survive process restarts or be shared across machines — use a
  durable queue.
- You only need a concurrency cap with nothing else — `p-limit` is smaller.
- You need distributed rate limiting enforced across replicas — you need a
  central store, not an in-memory counter.

## Alternatives

- sindresorhus/p-limit — same author, concurrency-only; use it when you don't
  need rate limiting, priorities, or events.
- SGrondin/bottleneck — heavier rate-limiter with Redis-backed clustering; use
  it when limits must hold across processes.
- taskforcesh/bullmq — Redis-backed persistent job queue; use it when jobs must
  survive restarts and run on servers.
- mcollina/fastq — very low-overhead in-process worker queue; use it when you
  want minimal allocations and no rate limiting.
- caolan/async — `async.queue` from the callback era; use it in legacy
  callback-style codebases not built around promises.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-10 | Initial release: promise queue with concurrency control.[^5] |
| 2.0 | 2018 | Interval-based rate limiting (`intervalCap` + `interval`). |
| 6.0 | 2020 | Priority queue, `AbortSignal`, TypeScript rewrite. |
| 7.0 | 2021 | ESM-only; CommonJS export dropped.[^2] |
| 8.0 | 2023 | Node version floor raised; dependency refresh. |
| — | 2024–2026 | Additive API: `strict` sliding-window mode, `onRateLimit`, `onPendingZero`, `runningTasks`, `setPriority`; declared feature complete.[^1] |

## References

[^1]: p-queue README — "the project is feature complete… we don't plan any
    further development" and the servers/job-queue note.
    https://github.com/sindresorhus/p-queue#readme
[^2]: p-queue README, Install — native ESM warning and CommonJS conversion note.
    https://github.com/sindresorhus/p-queue#install
[^3]: p-queue README, API — "`PQueue(options?)` … is an `EventEmitter3`
    subclass." https://github.com/sindresorhus/p-queue#pqueueoptions
[^4]: p-queue README, `.clear()` — warning that queued `add()` promises never
    settle after clear. https://github.com/sindresorhus/p-queue#clear
[^5]: Repository created 2016-10-28. https://github.com/sindresorhus/p-queue

## Tags

javascript, typescript, nodejs, promise, concurrency, rate-limiting, async, queue, esm, task-scheduler
