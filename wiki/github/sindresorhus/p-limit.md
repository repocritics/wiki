# sindresorhus/p-limit

> Run promise-returning and async functions with a cap on how many run at once.

[GitHub repo](https://github.com/sindresorhus/p-limit) ·
[License: MIT](https://github.com/sindresorhus/p-limit/blob/main/license)

## Overview

`p-limit` is a ~70-line utility that limits how many async tasks execute
concurrently. You wrap each task in a `limit()` call; the wrapper runs the task
immediately if fewer than `concurrency` tasks are active, otherwise it queues the
task and starts it when a slot frees up. It is one of the most-depended-upon
packages in the JavaScript ecosystem — pulled in transitively by countless build
tools, test runners, and API clients — precisely because concurrency capping is a
recurring need and the standard library offers no primitive for it. `Promise.all`
starts everything at once; `p-limit` is the throttle you put in front of it.

The package is deliberately minimal. It does not queue by priority, pause/resume,
report progress, or retry — it caps concurrency and nothing else. That narrow
scope is the defining design decision: adjacent packages by the same author
(`p-queue`, `p-map`, `p-all`) cover the richer cases, and `p-limit` stays small
enough to be a leaf dependency you never think about.

The one thing that reliably surprises people is that `p-limit` has been **pure
ESM since v4** (2021)[^1]. There is no CommonJS build. A CommonJS project that
does `require('p-limit')` gets `ERR_REQUIRE_ESM` and must either move to ESM,
use dynamic `import()`, or pin to v3. This is the single largest source of
friction in the package's issue history and the most common reason teams stay on
old majors.

## Getting Started

```sh
npm install p-limit
```

```js
import pLimit from 'p-limit';

const limit = pLimit(2); // at most 2 running at once

const urls = ['/a', '/b', '/c', '/d', '/e'];
const results = await Promise.all(
	urls.map(url => limit(() => fetch(url).then(r => r.json())))
);
```

`limit(fn, ...args)` returns the promise from `fn`, so the array you pass to
`Promise.all` looks exactly as it would without a limiter — only the execution
timing changes.

## Architecture / How It Works

The whole implementation is a counter plus a queue. `activeCount` tracks running
tasks; a queue holds deferred starters. When you call `limit(fn)`, the wrapper
either runs `fn` now (if `activeCount < concurrency`) or enqueues a closure that
will run it later. Each task, on settle, decrements the counter and dequeues the
next waiter. Because settlement is what advances the queue, a task that never
settles stalls every task behind it — there is no timeout.

The queue is `yocto-queue`, the package's sole runtime dependency[^2]. This
matters more than it looks: a naive implementation backs the queue with an array
and calls `Array.prototype.shift()`, which is O(n) because it re-indexes every
remaining element. `yocto-queue` is a linked list with O(1) enqueue and dequeue,
so `p-limit` stays linear even when tens of thousands of tasks are queued behind
a small concurrency limit.

Later majors added ergonomic surface without changing the core:

- **`limit.map(iterable, mapper)`** — sugar for
  `Promise.all(Array.from(iterable, (v, i) => limit(mapper, v, i)))`.
- **`limitFunction(fn, {concurrency})`** — a named export that returns a
  *single* self-limiting function, for when you want one callable whose own
  invocations are capped rather than a limiter shared across many functions.
- **`limit.clearQueue()`** — drops *pending* tasks (not running ones). With the
  `rejectOnClear` option, pending tasks reject with an `AbortError` instead of
  hanging forever unsettled.
- **`limit.concurrency`** — readable and writable at runtime; raising it
  immediately pulls more tasks off the queue.

Current versions require **Node.js ≥ 20** and ship only `index.js` +
`index.d.ts`[^3]. It also runs in browsers — there are no Node built-ins in the
code path.

## Production Notes

- **The deadlock footgun.** Calling the *same* `limit` from inside a task that is
  already governed by it can wedge the queue: the outer task holds a slot while
  waiting on an inner task that can never get one. The README warns about this
  explicitly[^4]. Use a separate limiter for nested work.
- **`clearQueue()` without `rejectOnClear` leaves promises unsettled.** If you
  `await Promise.all(tasks)` and then clear the queue, the cleared promises
  neither resolve nor reject and the `Promise.all` hangs. `rejectOnClear: true`
  (v7+) is the fix; on older versions, don't await tasks you might clear.
- **No cancellation of in-flight work.** `clearQueue()` only discards tasks that
  haven't started. Anything already running runs to completion. Pair with
  `AbortController` inside your task bodies if you need real cancellation.
- **Errors don't stop the queue.** A rejected task frees its slot and the next
  queued task starts. `Promise.all` rejects on the first failure, but the
  remaining tasks still execute in the background — use `Promise.allSettled` if
  you need every task accounted for.
- **The ESM tax.** Bumping across the v3→v4 boundary in a CommonJS codebase is a
  migration, not a version bump. Transitive dependents that couldn't move to ESM
  are the reason v3 still sees downloads years later.
- **It caps concurrency, not rate.** Two tasks at once is not "two per second."
  For requests-per-interval limiting use `p-throttle`; for a full queue with
  priorities and pause/resume use `p-queue`.

## When to Use / When Not

**Use when:**
- You have a fixed set of async tasks and want to bound simultaneous execution
  (API fan-out, DB writes, file I/O, scraping).
- You want the smallest possible dependency for this one job.
- You're already on ESM (or willing to be).

**Avoid when:**
- You need rate limiting over time, retries, priorities, or pause/resume — reach
  for `p-queue`, `p-throttle`, or `p-retry`.
- You're mapping over inputs and want concurrency *and* per-item results in one
  call — `p-map` is the better-fit primitive.
- You're locked to CommonJS and can't use dynamic `import()` — pin `p-limit@3`.

## Alternatives

- sindresorhus/p-queue — use instead when you need a real queue: priorities,
  pause/resume, timeouts, and introspection.
- sindresorhus/p-map — use instead when you're transforming an iterable and want
  bounded concurrency plus collected results (and `stopOnError`/`concurrency` in
  one call).
- sindresorhus/p-throttle — use instead when the limit is per *time window*
  (N calls per interval), not per number of in-flight tasks.
- sindresorhus/p-all — use instead when you have a fixed list of distinct
  functions to run with optional concurrency, rather than a shared limiter.
- supercharge/promise-pool — use instead when you want a fluent builder API with
  results/errors partitioning and don't mind a larger surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2016-10-21 | Initial release. |
| v2.0.0 | 2018-11 | Dropped older Node; internal cleanup. |
| v3.0.0 | 2020-06-06 | Switched internal queue to `yocto-queue` (O(1) dequeue). |
| v4.0.0 | 2021-08-12 | Pure ESM; CommonJS build removed[^1]. |
| v5.0.0 | 2023-11-01 | Added `limitFunction` named export. |
| v6.0.0 | 2024-07-04 | Added `limit.map()`; Node baseline raised. |
| v7.0.0 | 2025-08-14 | Options-object concurrency + `rejectOnClear`; Node ≥ 20[^3]. |
| v7.3.0 | 2026-02-03 | Current release at time of writing[^5]. |

## References

[^1]: Sindre Sorhus, "Get Ready For ESM" — context on the pure-ESM migration that
p-limit v4 is part of. https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c
[^2]: `yocto-queue` — tiny FIFO queue backing p-limit's pending list.
https://github.com/sindresorhus/yocto-queue
[^3]: `package.json` at v7.3.0 declares `"type": "module"` and
`"engines": { "node": ">=20" }`.
https://github.com/sindresorhus/p-limit/blob/main/package.json
[^4]: p-limit README — nested-limiter deadlock warning and API reference.
https://github.com/sindresorhus/p-limit#readme
[^5]: GitHub Releases for sindresorhus/p-limit.
https://github.com/sindresorhus/p-limit/releases

## Tags

javascript, nodejs, concurrency, promises, async-await, esm, rate-limiting, queue, throttle, utility, single-purpose
