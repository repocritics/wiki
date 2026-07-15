# Delgan/loguru

> Python logging with one pre-configured global logger and a single `add()` call for every sink, format, filter, and level.

[GitHub repo](https://github.com/Delgan/loguru) ·
[Documentation](https://loguru.readthedocs.io) ·
[License: MIT](https://github.com/Delgan/loguru/blob/master/LICENSE)

## Overview

Loguru is a logging library built around one premise: the standard library's
`logging` is correct but tedious, and most of that tedium is boilerplate you
configure before writing your first log line. Loguru ships one already-configured
`logger` that writes to `stderr` out of the box, and folds handler, formatter,
filter, and level configuration into a single `add()` function[^1]. First
released in 2017[^2], it is one of the most widely used third-party logging
libraries in Python, with roughly 24,000 GitHub stars.

The defining tradeoff is the global singleton. There is exactly one `logger`,
shared process-wide. That makes the API pleasant — you never construct or look
up a logger by name — but it also means Loguru does not model the hierarchical,
per-module logger tree that `logging` uses and that most frameworks (Django,
Celery, gunicorn) expect to configure. The ergonomics are excellent for
application and script authors and awkward for library authors, who are told
never to call `add()` and to gate logging behind `disable()`/`enable()`
instead[^1]. Loguru is also still a pre-1.0 (0.x) project despite years of
production use; the API is stable in practice, but 1.0 semantics are not
committed[^3].

## Getting Started

```bash
pip install loguru
```

```python
from loguru import logger

logger.debug("That's it, beautiful and simple logging!")

# One call configures a sink, its format, filter, and level:
logger.add(
    "app_{time}.log",
    rotation="500 MB",      # rotate when the file grows past 500 MB
    retention="10 days",    # delete rotated files older than 10 days
    compression="zip",      # compress on rotation
    level="INFO",
    enqueue=True,           # required for multiprocess safety / async sinks
)

logger.info("Structured too: {user} did {action}", user="alice", action="login")
```

## Architecture / How It Works

Every log call assembles a **record** — a dict of `time`, `level`, `message`,
`file`, `line`, `function`, `extra`, `exception`, etc. — and dispatches it to
every registered **sink** whose level and filter accept it. A sink can be a
file path (string), a file-like object, a plain callable, a coroutine function,
or a stdlib `logging.Handler`[^1]. `add()` returns an integer id; `remove(id)`
tears the sink down, and `logger.remove()` with no argument drops the default
`stderr` handler for a clean slate.

Contextual state layers on top of the record's `extra` dict. `bind(**kw)` returns
a child logger that injects those keys into every record; `contextualize(**kw)`
does the same temporarily using `contextvars`, so it is coroutine- and
thread-local; `patch(fn)` lets you mutate each record dynamically. `opt(...)`
returns a one-shot modified logger for per-message behavior — lazy evaluation
(`opt(lazy=True)`), colorized markup (`opt(colors=True)`), raw output, or
adjusting `depth` so the reported caller is correct through wrapper functions.

Two internals drive most of Loguru's identity. **Colorized tracebacks with
variable values** come from a `better_exceptions`-style frame walker that
introspects locals at each stack level and renders them inline (`diagnose=True`).
**Async / multiprocess safety** is opt-in via `enqueue=True`, which pushes
records onto a `multiprocessing`-backed queue drained by a background thread, so
that writes are serialized across processes and the calling thread never blocks
on the sink. Sinks are thread-safe by default via locks; they are *not*
multiprocess-safe without `enqueue`.

## Production Notes

**`diagnose=True` is a data-leak footgun.** The variable-value traceback feature
is on by default. In production those annotated frames can dump secrets, tokens,
and PII into your logs. The maintainer documents this explicitly: set
`diagnose=False` for any handler whose output leaves the developer's machine[^4].

**Interop with standard `logging` is the biggest real-world friction.** Loguru
does not participate in the stdlib logger tree, so third-party libraries logging
through `logging` do not reach your Loguru sinks unless you install an
`InterceptHandler` that forwards records — a ~15-line boilerplate every serious
Loguru deployment copies[^1]. Going the other direction (Loguru → stdlib) needs a
`PropagateHandler`.

pytest's `caplog` fixture does not capture Loguru for the same reason; you must
route Loguru into the `caplog` handler or the assertions silently see nothing[^5].

**Multiprocess and forking servers need `enqueue=True`.** gunicorn/uvicorn
workers, `multiprocessing.Pool`, and Celery prefork will otherwise interleave or
corrupt file writes. Enqueuing adds a background thread and a queue you should
drain with `logger.complete()` / `logger.remove()` at shutdown to avoid losing
buffered records.

**The global singleton leaks state across tests.** Because there is one logger,
sinks added in one test persist into the next unless you `remove()` them; the
common pattern is an autouse fixture that snapshots and restores handlers.

**It is not "10x faster than logging."** An old README claim to that effect was
struck through by the maintainer, and a promised C-accelerated core has not
shipped[^6]. Treat Loguru as comparable to or somewhat slower than stdlib
`logging` on the hot path; use `opt(lazy=True)` to defer expensive message
construction rather than assuming the library is free.

**Library authors: never call `add()`.** A library that adds sinks hijacks the
consumer's global logger. Use `logger.disable("your_package")` and let the
application `enable()` it[^1].

## When to Use / When Not

**Use when:**
- You are writing an application, service, or script and want good logging with
  near-zero setup.
- You want rotation, retention, compression, colorized diagnostic tracebacks, and
  JSON serialization (`serialize=True`) without wiring handlers by hand.
- You value ergonomics and a single obvious way to configure output.

**Avoid when:**
- You are writing a library meant for broad consumption — forcing Loguru on
  downstream users is intrusive; stdlib `logging` is the neutral default.
- Your stack (Django/ops tooling) expects `logging.config.dictConfig` and
  per-module logger levels; you would be fighting the framework.
- You need audited, structured-first logging with explicit processors and no
  hidden global state — reach for structlog.
- Logging throughput is genuinely performance-critical.

## Alternatives

- python/cpython (`logging`) — use the standard library when you need ecosystem
  interop, per-module configuration, or you are shipping a library.
- hynek/structlog — use when you want structured/JSON-first logging with explicit
  context binding and a configurable processor pipeline, no global singleton.
- madzak/python-json-logger — use when you only need stdlib `logging` to emit JSON
  and want minimal disruption.
- microsoft/picologging — use when you need a near-drop-in, faster reimplementation
  of stdlib `logging` on the hot path.
- itamarst/eliot — use when you want causal, action/tree-structured logs rather
  than a flat message stream.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.1 | 2017-12 | First PyPI release[^2]. |
| 0.4.x | 2019 | Coroutine sinks, `complete()` for async draining[^3]. |
| 0.5.0 | 2020 | `contextualize()`, improved multiprocessing handling[^3]. |
| 0.6.0 | 2022 | Dropped Python 3.5 support; typing improvements[^3]. |
| 0.7.0 | 2023 | Environment-variable defaults; packaged type hints[^3]. |
| 0.7.3 | 2024-12 | Maintenance release; latest 0.x line[^3]. |

## References

[^1]: Loguru README and "Overview" — the `add()` sink model, `bind`/`contextualize`/`patch`, and library `disable()`/`enable()` guidance. https://github.com/Delgan/loguru#readme
[^2]: Loguru on PyPI — release history. https://pypi.org/project/loguru/#history
[^3]: Loguru changelog. https://loguru.readthedocs.io/en/stable/project/changelog.html
[^4]: Loguru docs, "Security considerations when using Loguru" — `diagnose=True` can leak sensitive data. https://loguru.readthedocs.io/en/stable/resources/recipes.html#security-considerations-when-using-loguru
[^5]: Loguru docs, "Testing logging" — pytest `caplog` does not capture Loguru without a fixture. https://loguru.readthedocs.io/en/stable/resources/migration.html
[^6]: Loguru README — the struck-through "10x faster than built-in logging" claim and the pending C implementation. https://github.com/Delgan/loguru#readme

## Tags

python, logging, logger, observability, structured-logging, stderr, tracebacks, developer-tooling, library, single-global-logger
