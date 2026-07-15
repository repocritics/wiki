# agronholm/anyio

> A structured-concurrency layer that lets one async codebase run unmodified on either asyncio or Trio.

[GitHub repo](https://github.com/agronholm/anyio) ·
[Documentation](https://anyio.readthedocs.io/) ·
[License: MIT](https://github.com/agronholm/anyio/blob/master/LICENSE)

## Overview

AnyIO is a Python library, maintained primarily by Alex Grönholm, that implements Trio-style structured concurrency (SC) on top of the `asyncio` event loop, while also running natively on Trio[^1]. Code written against AnyIO's API — task groups, cancel scopes, memory object streams, worker-thread helpers — behaves the same regardless of which backend the program is launched on. The backend is chosen once, at `anyio.run(...)` time (or by the enclosing loop), not woven through the call sites.

The core value proposition is portability plus safety. Library authors can target AnyIO instead of raw `asyncio` and thereby support Trio users for free, and gain structured concurrency (guaranteed child-task cleanup, deadline-based cancellation, exception propagation as groups) that stock asyncio historically lacked. In practice most AnyIO code runs on the asyncio backend; Trio compatibility is the differentiator that justifies the abstraction rather than the common deployment target.

AnyIO is best known as invisible infrastructure. Starlette and FastAPI use it for their thread-pool and task handling, and HTTPX uses it for its async transport[^2]. Because of this, a large share of Python async web apps depend on AnyIO transitively without their authors writing a line of AnyIO themselves. The defining tension is that AnyIO reimposes Trio's stricter cancellation and error semantics on asyncio — cleaner and safer, but subtly different from the asyncio behavior developers may expect.

## Getting Started

```bash
pip install anyio
```

```python
import anyio

async def worker(name, results):
    await anyio.sleep(0.1)
    results.append(name)

async def main():
    results = []
    # Task group: the with-block does not exit until every child finishes;
    # if one raises, the rest are cancelled and errors surface as a group.
    async with anyio.create_task_group() as tg:
        for i in range(3):
            tg.start_soon(worker, f"task-{i}", results)
    print(results)

# Runs identically on asyncio (default) or Trio: backend="trio"
anyio.run(main)
```

## Architecture / How It Works

AnyIO exposes a single API and dispatches to a backend module at runtime. On Trio the calls are thin pass-throughs to Trio's native primitives. On asyncio, AnyIO rebuilds Trio's model on top of the event loop:

- **Cancel scopes** — the unit of cancellation. `anyio.move_on_after(seconds)` and `anyio.fail_after(seconds)` create scopes with deadlines. On asyncio, AnyIO tracks scope nesting itself rather than relying on task-level cancellation, so cancellation is delivered at `await` points and is level-triggered within a scope, matching Trio semantics rather than asyncio's edge-triggered `CancelledError`.
- **Task groups** — `create_task_group()` is Trio's "nursery." The block cannot exit until all `start_soon`/`start` children complete; an unhandled child exception cancels siblings and propagates. This is the same guarantee stdlib `asyncio.TaskGroup` added in Python 3.11, which AnyIO predates.
- **Memory object streams** — `create_memory_object_stream()` replaces `asyncio.Queue` with a closeable, backpressure-aware send/receive pair that participates in cancellation and SC cleanup.
- **Thread bridging** — `to_thread.run_sync()` offloads blocking calls to a worker thread; `from_thread.run()` and blocking portals (`start_blocking_portal()`) let synchronous code call back into the async world across the thread boundary.
- **Networking** — high-level TCP/UDP/UNIX socket APIs with a Happy Eyeballs implementation for TCP connects, plus TLS streams, presented uniformly across backends.

Since AnyIO 4.0 (2023) task groups raise native `ExceptionGroup` (PEP 654); on Python versions without it, the `exceptiongroup` backport is a dependency[^3]. AnyIO ships its own pytest plugin that parametrizes async tests and fixtures over backends via the `anyio_backend` fixture, so a suite can run once against asyncio and once against Trio.

## Production Notes

- **Cancellation is not asyncio's cancellation.** AnyIO's cancel scopes are level-triggered: while a scope is cancelled, every checkpoint inside it keeps raising until control leaves the scope. Code that catches `CancelledError` to run cleanup and then continues — a common asyncio pattern — can loop or misbehave. Shielding must be done through AnyIO's `CancelScope(shield=True)`, not asyncio's `shield()`.
- **Exception groups are a breaking mental model.** Once a task group can raise `ExceptionGroup`, `try/except SomeError` around the group no longer catches the inner error directly. You need `except*` (3.11+) or to catch the group and inspect it. Upgrading a codebase to AnyIO 4 surfaced this widely[^3].
- **You inherit AnyIO transitively.** If you depend on FastAPI/Starlette or HTTPX, AnyIO is already in your tree, and its major-version bumps can change thread-pool or cancellation behavior underneath frameworks you use[^2]. Pin and read the changelog on major upgrades.
- **Overhead is small but real.** The asyncio backend adds bookkeeping (scope tracking, checkpoint wrapping) over raw asyncio. For most I/O-bound workloads this is negligible; in extremely hot task-spawn loops it is measurable. Raw asyncio remains marginally faster.
- **Trio is the minority backend.** The overwhelming majority of deployments run on asyncio. Trio-path bugs get found later simply because fewer users exercise them; if you rely on the Trio backend, run your own tests against it rather than assuming parity.
- **`start_soon` swallows nothing but reorders when.** Because children run to completion at block exit, an exception raised early may only surface when the task group unwinds, which can be later than intuition suggests.

## When to Use / When Not

**Use when:**
- You are writing a library and want it usable from both asyncio and Trio applications.
- You want structured concurrency, deadline-based cancellation, and group error handling on asyncio, especially targeting Python versions before 3.11's `asyncio.TaskGroup`.
- You are already inside the Starlette/FastAPI/HTTPX ecosystem and want to work with the concurrency model they use rather than against it.

**Avoid when:**
- You are asyncio-only on Python 3.11+ and are content with stdlib `asyncio.TaskGroup` / `asyncio.timeout` — you may not need the extra dependency.
- You want Trio's model with none of asyncio's baggage: use Trio directly.
- You have a heavily asyncio-idiomatic codebase and team; AnyIO's stricter cancellation semantics are a retraining cost.

## Alternatives

- python-trio/trio — the native structured-concurrency runtime AnyIO emulates; use it directly when you don't need asyncio interop or its ecosystem.
- python/cpython (stdlib asyncio) — Python 3.11+ ships `asyncio.TaskGroup` and `asyncio.timeout`; use plain asyncio when you're asyncio-only and don't need Trio portability.
- dabeaz/curio — the earlier coroutine-runtime experiment that inspired this space; use only for study, as it is effectively unmaintained.
- florimondmanca/aiometer — task-level concurrency limiting and rate control; use alongside or instead when your need is bounded fan-out rather than a full SC layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2018-08 | First commits by Alex Grönholm[^1]. |
| 2.0 | 2020 | curio backend removed; asyncio + Trio only. |
| 3.0 | 2021-04 | Several previously-sync APIs became async; cancellation and stream APIs reworked[^4]. |
| 4.0 | 2023 | Native PEP 654 `ExceptionGroup`; task groups raise groups; `exceptiongroup` backport dependency on older Pythons[^3]. |

## References

[^1]: AnyIO README and repository. https://github.com/agronholm/anyio
[^2]: AnyIO is a dependency of Starlette/FastAPI (thread pool, task handling) and HTTPX (async transport). https://www.starlette.io/ and https://www.python-httpx.org/
[^3]: AnyIO documentation, "Version history" — 4.0 exception group changes. https://anyio.readthedocs.io/en/stable/versionhistory.html
[^4]: AnyIO documentation, migration and version history. https://anyio.readthedocs.io/en/stable/migration.html

## Tags

python, asyncio, trio, structured-concurrency, concurrency, async-await, networking, library, task-groups, cancellation
