# python-trio/trio

> An async/await-native concurrency library for Python built around structured concurrency — every task has a parent scope, and errors and cancellation propagate along that tree.

[GitHub repo](https://github.com/python-trio/trio) ·
[Official website](https://trio.readthedocs.io) ·
[License: MIT OR Apache-2.0](https://github.com/python-trio/trio/blob/main/LICENSE)

## Overview

Trio is an alternative to Python's standard `asyncio`, started by Nathaniel J. Smith in 2017[^1]. It shares the `async`/`await` syntax and the "single-threaded event loop" execution model, but rejects `asyncio`'s API surface almost entirely. Its organizing idea is *structured concurrency*: you cannot spawn a background task into the void. Every concurrent task must be opened inside a *nursery* (`trio.open_nursery()`), and the `async with` block that owns the nursery will not exit until every child task inside it has finished[^2]. This makes concurrency lexically scoped the same way a function body is — the shape of the code matches the lifetime of the tasks.

The payoff is that two hard problems — error propagation and cancellation — become tractable. If a child task raises, the exception surfaces at the nursery boundary instead of being silently swallowed by a fire-and-forget task. Cancellation is delivered through *cancel scopes* with deadlines, and it works by injecting a `Cancelled` exception at the next checkpoint (every `await` of a Trio primitive), so it composes with normal `try`/`finally` cleanup instead of fighting it. Trio's design directly influenced the wider Python ecosystem: its `MultiError` became the model for `ExceptionGroup` and the `except*` syntax standardized in PEP 654 (Python 3.11)[^3], and `asyncio` later grew a `TaskGroup` that borrows the nursery concept.

The defining tension is ecosystem gravity. Trio is a cleaner core, but `asyncio` is in the standard library, so the overwhelming majority of async libraries (database drivers, HTTP servers, cloud SDKs) target `asyncio` first or only. Trio's answer is `anyio`, a compatibility layer that lets library authors write once against a Trio-shaped API and run on either backend — but adopting Trio in an existing `asyncio` stack still means checking, per dependency, whether it works.

## Getting Started

```bash
pip install trio
# requires Python 3.10+ (CPython or a maintained PyPy3)
```

```python
import trio

async def child(name, seconds):
    print(f"{name} starting")
    await trio.sleep(seconds)
    print(f"{name} done")

async def main():
    # The nursery block does not exit until BOTH children finish.
    async with trio.open_nursery() as nursery:
        nursery.start_soon(child, "task-A", 2)
        nursery.start_soon(child, "task-B", 1)
    print("all children complete")

trio.run(main)
```

```python
# Cancellation with a deadline — no task leaks past the timeout.
async def fetch_with_timeout():
    with trio.move_on_after(5):        # cancel scope, 5s budget
        await slow_network_call()
    # control resumes here whether it finished or was cancelled
```

## Architecture / How It Works

At the center is a single event loop started by `trio.run()`. There is exactly one supported loop implementation — Trio does not expose a pluggable loop policy the way `asyncio` does — which removes a whole class of "which loop am I on" bugs but also means you do not swap in `uvloop`.

Key concepts, in the order they matter:

- **Nurseries.** `trio.open_nursery()` yields an object with `start_soon()` and `start()`. The nursery is the *only* way to run tasks concurrently; there is no top-level `create_task()` that outlives its caller. When the `async with` block exits, it waits (in an implicit `finally`) for all children.
- **Cancel scopes.** Timeouts and cancellation are values, not exceptions you catch by type at call sites. `trio.move_on_after`, `trio.fail_after`, and a manually created `trio.CancelScope` define a region; cancelling the scope delivers `trio.Cancelled` at the next checkpoint inside it. Scopes nest, and cancellation of an outer scope propagates inward.
- **Checkpoints.** Every Trio operation that can block is a checkpoint: it is where cancellation can be delivered and where the scheduler can switch tasks. Trio guarantees checkpoints have *both* properties, which makes it possible to reason about — and test — cancellation and fairness. Code that does CPU work without hitting a checkpoint will starve other tasks, same as any cooperative scheduler.
- **`MultiError` / `ExceptionGroup`.** Because a nursery can have several children fail at once, Trio needed a way to raise multiple exceptions together. This mechanism was upstreamed into the language as `ExceptionGroup`; modern Trio raises standard exception groups[^3].
- **Instrumentation and testing.** Trio ships a first-class `trio.testing` module with a virtual clock (`autojump_clock`) so time-based tests run instantly and deterministically, plus an `Instrument` API for tracing scheduler events.

Interop is deliberate, not implicit. `trio-asyncio` runs an `asyncio` loop *inside* a Trio program so you can call `asyncio`-only libraries. `anyio` goes the other direction at the library level, abstracting over both backends. Neither is free — each adds a translation layer with its own edge cases around cancellation semantics.

## Production Notes

- **The ecosystem is the constraint, not the core.** Trio itself is mature and stable, but you must vet each dependency. Popular Trio-native or `anyio`-based options exist (`httpx` for HTTP clients, `asyncpg` via wrappers, `hypercorn`/`quart-trio` for serving), but many mainstream async libraries are `asyncio`-only and require `trio-asyncio` to use.
- **Cancellation is exception-based, so cleanup code must be checkpoint-aware.** A `finally` block that itself needs to `await` can be interrupted by cancellation. Trio provides `trio.CancelScope(shield=True)` to protect critical cleanup, but forgetting to shield is a real footgun that can leave resources half-released.
- **No implicit background tasks.** Patterns that assume fire-and-forget (`asyncio.ensure_future` with no owner) do not translate. You must decide *which nursery owns* a long-lived task — often a nursery passed down from `main`. This is more upfront design but eliminates leaked-task bugs.
- **Still versioned 0.x.** Trio has never declared a 1.0 and reserves the right to make breaking changes, though in practice they are infrequent and well-telegraphed; the maintainers point users to issue #1 for advance notice of compatibility changes[^4]. Pin your version and read the changelog before upgrading.
- **CPU-bound work blocks the loop.** As with any single-threaded async runtime, offload blocking or CPU-heavy calls with `trio.to_thread.run_sync()` (or a process pool) — otherwise one task stalls all of them.
- **Platform support** is CPython/PyPy3 on Linux, macOS, Windows, and FreeBSD; dependencies are pure Python except CFFI on Windows (wheels available, no C compiler needed).

## When to Use / When Not

**Use when:**
- You are starting a new async codebase and want correctness-by-construction: no leaked tasks, exceptions that always surface, timeouts that compose.
- Your concurrency is complex enough (fan-out, supervision, partial cancellation) that `asyncio`'s manual task management becomes error-prone.
- You are writing a *library* and can target `anyio`, getting both backends for the price of one.

**Avoid when:**
- You depend on `asyncio`-only libraries that you cannot afford to wrap through `trio-asyncio`.
- Your team or deployment expects the standard-library default and stack-overflow-answer ecosystem that `asyncio` has.
- You need `uvloop`-level raw throughput with a drop-in loop swap.

## Alternatives

- python/cpython (`asyncio`) — use instead when standard-library inclusion and the largest driver/library ecosystem matter more than a clean API.
- agronholm/anyio — use as a layer *on top of* Trio or asyncio when you are writing a library that must support both backends.
- dabeaz/curio — use to study the ideas in their most minimal form; Curio is the research predecessor that inspired Trio but is far less active.
- MagicStack/uvloop — use when you want a faster drop-in `asyncio` event loop and are staying within the `asyncio` model.
- twisted/twisted — use for mature, callback/Deferred-style networking with decades of battle-tested protocol implementations.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017 | First public release by Nathaniel J. Smith; structured-concurrency model and nurseries[^1]. |
| 0.x | 2017–present | Long 0.x line; APIs stabilized in practice, breaking changes rare but reserved[^4]. |
| — | 2018 | "Notes on structured concurrency" essay and PyCon talk popularize the model[^2]. |
| — | 2022 | `MultiError` idea standardized as `ExceptionGroup` / `except*` in PEP 654, Python 3.11[^3]. |
| current | ongoing | Requires Python 3.10+; still pre-1.0 by design. |

## References

[^1]: Nathaniel J. Smith, "Announcing Trio" — 2017. https://vorpus.org/blog/announcing-trio/
[^2]: Nathaniel J. Smith, "Notes on structured concurrency, or: Go statement considered harmful" — 2018. https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/
[^3]: PEP 654 — Exception Groups and except*. https://peps.python.org/pep-0654/
[^4]: Trio README and issue #1 (compatibility notices). https://github.com/python-trio/trio/issues/1

## Tags

python, async, await, structured-concurrency, concurrency, asyncio-alternative, io, networking, event-loop, cancellation, library
