# jd/tenacity

> General-purpose retrying library for Python — compose stop, wait, and retry policies as objects and attach them with a decorator.

[GitHub repo](https://github.com/jd/tenacity) ·
[Official website](https://tenacity.readthedocs.io) ·
[License: Apache-2.0](https://github.com/jd/tenacity/blob/main/LICENSE)

## Overview

Tenacity is a retry library for Python by Julien Danjou. It began as a fork of Ray Holder's `retrying`, which had gone unmaintained and carried longstanding bugs; tenacity is deliberately not API-compatible with its ancestor and reworked the design around composable policy objects[^1]. As of 2026 it is the default answer to "how do I add retries in Python" and a transitive dependency of a large part of the ecosystem (it is pulled in by many HTTP clients, cloud SDKs, and LangChain-adjacent tooling), which is why its ~8.7k stars understate its actual reach.

The defining idea is that a retry policy is decomposed into orthogonal, first-class objects: a `stop` condition (when to give up), a `wait` strategy (how long to sleep between attempts), and a `retry` predicate (what counts as a failure worth retrying). Each is an object you can combine with operators — `stop_after_delay(10) | stop_after_attempt(5)`, `wait_fixed(3) + wait_random(0, 2)` — rather than a fixed set of decorator keyword arguments. This composability is the reason tenacity displaced simpler decorators: the same primitives cover a trivial `@retry` and a production policy with exponential backoff, jitter, result-based retries, and structured logging.

The tension is that this generality comes with a large surface area and easy-to-misconfigure defaults. A bare `@retry` retries *forever* with *no wait* on *any* `Exception` — fine for a demo, a denial-of-service against your own upstream in production. Tenacity gives you every knob but picks unsafe defaults, so correct usage is always explicit.

## Getting Started

```bash
pip install tenacity
```

```python
from tenacity import (
    retry, stop_after_attempt, wait_random_exponential,
    retry_if_exception_type, before_sleep_log,
)
import logging

logger = logging.getLogger(__name__)

@retry(
    retry=retry_if_exception_type(ConnectionError),
    stop=stop_after_attempt(5),
    wait=wait_random_exponential(multiplier=1, max=60),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
def fetch(url: str) -> bytes:
    ...  # raises ConnectionError on transient failure
```

This retries only on `ConnectionError`, up to 5 attempts, with exponential backoff plus jitter capped at 60s, logs each retry, and re-raises the original exception (not `RetryError`) on final failure.

## Architecture / How It Works

The core is the `Retrying` object (and `AsyncRetrying` for coroutines). The `@retry` decorator is a thin wrapper that constructs a `Retrying` from its keyword arguments and drives the wrapped call through it. Each invocation runs a loop that, per attempt, builds a `RetryCallState` carrying `attempt_number`, an `outcome` (a `concurrent.futures.Future` holding the last return value or exception), timing (`seconds_since_start`, `idle_for`, `start_time`), and the bound function/args.

Every policy component is a callable taking that `retry_state` and returning a decision:

- **`stop`** returns `bool` — should we stop. Built-ins: `stop_after_attempt`, `stop_after_delay`, `stop_before_delay`, `stop_never`.
- **`wait`** returns seconds to sleep. Built-ins: `wait_fixed`, `wait_random`, `wait_exponential`, `wait_random_exponential`, `wait_incrementing`, `wait_chain`, `wait_combine`. `+` combines waits; the exponential/jitter variants are the ones you want against remote services.
- **`retry`** returns whether this outcome is retryable. Built-ins split into exception-based (`retry_if_exception_type`, `retry_if_not_exception_type`, `retry_if_exception_message`) and result-based (`retry_if_result`, `retry_if_not_result`), combinable with `retry_any`/`retry_all` or the `|`/`&` operators.

Because these are plain objects, the same policy drives three usage shapes: the `@retry` decorator, direct `Retrying(...)` calls, and — for retrying an inline block without extracting a function — the iterator/context-manager form (`for attempt in Retrying(...): with attempt:`). The `with attempt:` block captures exceptions and feeds them back into the loop. `AsyncRetrying` mirrors this with `async for`.

Async support is runtime-agnostic: `retry` works on asyncio, Trio, and Tornado coroutines, and sleeps asynchronously. The sleep function is injectable (`@retry(sleep=trio.sleep)`), which is how non-asyncio loops are supported without a hard dependency on any of them.

## Production Notes

**The default `@retry` is a footgun.** No `stop`, no `wait`, `retry` on all exceptions means infinite, tight-loop retries that can hammer a failing dependency and mask real errors. Always set an explicit `stop` and a backoff `wait`, and narrow the `retry` predicate to genuinely transient errors — retrying a `ValueError` or a 4xx client error just wastes time.

**Prefer `reraise=True`.** By default, exhausting retries raises `RetryError`, and the underlying exception is buried in the *middle* of the traceback. `reraise=True` surfaces the original exception at the end where callers and error trackers expect it. Without it, `except SpecificError` clauses upstream silently stop matching.

**Use jitter for anything distributed.** Plain `wait_exponential` synchronizes retries across many clients into a thundering herd. `wait_random_exponential` (or `wait_fixed + wait_random`) spreads them out. This matters as soon as more than one process shares an upstream.

**Testing waits is a common pain.** The live `wait` sleeps for real, so naive tests are slow. The supported pattern is patching the write-only `.retry` attribute on the decorated function (`mock.patch.object(fn.retry, "wait", wait_fixed(0))`), or setting `enabled=False` to bypass retrying entirely. Read statistics from the separate `.statistics` attribute — `.retry` is write-only.

**Generators are not retried.** `@retry` wraps the *call*, not iteration. Decorating a generator or async generator retries only the initial call that returns the generator object; exceptions raised while consuming it are never caught. This surprises people wrapping streaming APIs.

**Observability.** Each decorated function exposes a `.statistics` dict (attempt count, idle time, etc.), and `before` / `after` / `before_sleep` callbacks hook the lifecycle — `before_sleep` is the right place to reconnect, refresh a token, or emit a metric before the next attempt. `before_sleep_log`, `after_log`, and `before_log` are built-in logging helpers.

**Version churn is low but real.** The object-based API has been stable since the 8.x line (2021); 9.0.0 (2024) dropped Python 3.7 and did minor cleanup. Pin a lower bound but upgrades are usually painless. It ships type hints and is broadly typed.

## When to Use / When Not

**Use when:**
- You need retries with backoff/jitter around network calls, flaky I/O, or eventually-consistent APIs.
- You want one policy vocabulary across sync functions, async coroutines, and inline blocks.
- You need result-based retries (retry until a poll returns "ready"), not just exception-based.
- You want retry statistics and lifecycle hooks for logging/metrics.

**Avoid when:**
- You only need "retry an HTTP request 3 times" — `urllib3`'s built-in `Retry` (already inside `requests`/`httpx` transports) handles connection-level retries with pooling and no extra dependency.
- Your framework already owns retries (Celery task `retry`, message-queue redelivery, a service mesh) — layering tenacity on top double-retries.
- You want an opinionated, hard-to-misconfigure wrapper — tenacity's defaults are unsafe by design; consider a thin layer over it.

## Alternatives

- hynek/stamina — an opinionated production wrapper *around tenacity itself*: safe defaults, typing, async, and Prometheus hooks. Use when you want tenacity's engine without its footguns.
- litl/backoff — decorator-based backoff with a simpler API and lighter footprint. Use for straightforward exception/predicate retries; note it is far less actively maintained.
- urllib3/urllib3 — its `Retry` object does transport-level HTTP retries with connection pooling. Use when you are specifically retrying HTTP requests, not arbitrary code.
- invl/retry — tiny retry decorator for scripts. Use when you want the absolute minimum and don't need composability; effectively unmaintained.

## History

| Version | Date | Notes |
|---------|------|-------|
| (fork) | 2016-08 | Forked from rholder/retrying; reworked around composable policy objects[^1]. |
| 5.0.0 | 2018-07-25 | Object-based `stop`/`wait`/`retry` API maturing. |
| 6.0.0 | 2019-11-06 | Async support (`AsyncRetrying`); Python 3-only. |
| 8.0.0 | 2021-07-07 | Stabilized modern API; type hints. |
| 8.2.0 | 2023-02-06 | Additional wait/retry helpers, fixes. |
| 8.5.0 | 2024-07-05 | Late 8.x maintenance release. |
| 9.0.0 | 2024-07-29 | Dropped older Python versions; cleanup[^2]. |
| 9.1.4 | 2026-02-07 | Current maintenance release[^2]. |

## References

[^1]: "Python retrying library" / tenacity README — origin as a fork of `rholder/retrying`. https://github.com/jd/tenacity and https://julien.danjou.info/python-tenacity/
[^2]: Tenacity releases and changelog. https://github.com/jd/tenacity/releases

## Tags

python, retry, backoff, resilience, error-handling, decorator, async, asyncio, exponential-backoff, jitter, library
