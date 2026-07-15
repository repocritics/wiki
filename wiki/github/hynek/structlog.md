# hynek/structlog

> Structured logging for Python built around a single idea: a log call is a dictionary passed through a list of functions.

[GitHub repo](https://github.com/hynek/structlog) ·
[Official website](https://www.structlog.org/) ·
[License: MIT OR Apache-2.0](https://github.com/hynek/structlog/blob/main/COPYRIGHT)

## Overview

structlog is a logging library written and maintained by Hynek Schlawack, in
continuous production use since 2013[^1]. Its premise is that a log event is not
a formatted string but a dictionary of key/value pairs (an *event dict*), and
that everything you want to do to a log entry — add a timestamp, add the log
level, merge in request-scoped context, render to JSON — is just a function that
takes that dict and returns a new one. A *processor chain* is an ordered list of
these functions; the last one turns the dict into bytes or a string.

The defining tension is flexibility versus a configuration learning curve.
Because structlog does not impose an output format, a transport, or a filtering
model, a first-time user faces a `configure()` call and a processor list before
anything useful happens, and the "right" processor order is not obvious. In
exchange, teams that outgrow the standard library's `logging` module get a system
where console-pretty output in development and machine-parseable JSON in
production are two different processor lists over identical call sites.

structlog does not have to replace the standard library. It can render output
itself (via lightweight `PrintLogger`/`WriteLogger` sinks), or it can format the
event dict and hand it off to stdlib `logging` for routing to files, syslog, or
third-party handlers. Which of these two integration modes a project picks is the
single most consequential decision when adopting it, and the source of most
confusion.

## Getting Started

```bash
pip install structlog
# optional pretty console colors:  pip install structlog[dev]  (pulls in rich/better-exceptions)
```

```python
import structlog

log = structlog.get_logger()

log = log.bind(request_id="abc123", user="tom")
log.info("user_action", action="checkout", cart_size=3)
# console (default): 2026-07-15T10:12:00Z [info  ] user_action  action=checkout cart_size=3 request_id=abc123 user=tom
```

```python
# Production: emit JSON instead, configured once at startup.
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)
structlog.get_logger().warning("disk_low", free_mb=412)
# {"free_mb": 412, "event": "disk_low", "level": "warning", "timestamp": "2026-07-15T10:12:00Z"}
```

## Architecture / How It Works

The core object is the *bound logger*. `get_logger()` returns a lazy proxy; the
first method call materializes a concrete bound logger and (by default) caches it
on the module-level logger for reuse[^2]. `bind()` returns a *new* bound logger
carrying an extended context dict — bound loggers are immutable, so context does
not leak between call sites the way a mutated singleton would.

When you call `log.info("event", **kw)`, structlog builds an event dict from the
bound context plus the call-site keywords, then runs it left-to-right through the
configured processor list. Each processor receives `(logger, method_name,
event_dict)` and returns a dict (or raises `DropEvent` to discard it). The final
processor is a *renderer* — `JSONRenderer`, `ConsoleRenderer`, `KeyValueRenderer`
(logfmt-style) — that returns the object actually passed to the underlying logger.

Two integration topologies exist:

1. **Standalone.** structlog owns the whole pipeline and writes via `PrintLogger`
   or `WriteLogger`. Simple, fast, no stdlib involvement.
2. **Stdlib-wrapped.** structlog formats the event dict and calls stdlib
   `logging`; a `ProcessorFormatter` on the stdlib side runs the renderer so that
   logs from *non*-structlog libraries flow through the same output format[^3].
   This is the correct choice when third-party packages also log, but the
   dual-configuration (structlog processors + `logging.config` handlers) is the
   most error-prone part of the library.

Request-scoped context is handled with `contextvars`: `bind_contextvars()` /
`merge_contextvars` attach per-task data that survives across `await` points
without being threaded through function arguments. Filtering is done by choosing a
bound logger class via `make_filtering_bound_logger(level)`, which drops below-level
calls before the processor chain runs — the level check is not a processor, which
is why it is cheap.

## Production Notes

**Processor order is load-bearing and silent when wrong.** `add_log_level` must
run before anything that reads `level`; the renderer must be last; `format_exc_info`
/ `ExceptionRenderer` must precede the renderer or tracebacks vanish. There is no
validation — a misordered chain produces subtly wrong output, not an error.

**`cache_logger_on_first_use=True` freezes configuration.** Caching the bound
logger on first use is recommended for performance but means a later
`configure()` call will not affect already-cached loggers. In tests, use
`configure` before importing modules under test, or `reset_defaults()` between
cases.

**Native `logging` interop is a second configuration surface.** To make stdlib
and structlog logs share a format you configure `ProcessorFormatter` with
`foreign_pre_chain` for records that did not originate in structlog. Getting
timestamps and levels consistent across both sources takes iteration; expect to
debug double-rendered dicts and missing keys.

**Async.** structlog offers `await log.ainfo(...)` (`adebug`, `awarning`, etc.)
that run the processor chain in a thread executor so a slow renderer or sink does
not block the event loop[^4]. The synchronous methods still work in async code;
the `a*` variants matter only when a processor does blocking I/O.

**Performance.** structlog is fast when configured for it: filtering bound loggers
short-circuit below-level calls, and `cache_logger_on_first_use` avoids repeated
setup. The slow paths are heavy processors (exception formatting, `CallsiteParameterAdder`
which inspects the stack) and `ConsoleRenderer`, which is for humans, not hot loops.
Use `JSONRenderer` (or `orjson`) in production.

**Upgrade note.** structlog dropped Python 2 and gained `contextvars` + async in
the 20.x line[^5]; APIs have been stable since. It uses CalVer, so a jump from
24.x to 25.x is a normal release, not a semantic-versioned break — read the
CHANGELOG rather than inferring risk from the version delta.

## When to Use / When Not

**Use when:**
- You want structured (JSON/logfmt) logs and per-request context without hand-building dicts at every call site.
- You need pretty colorized console output in dev and machine-parseable output in prod from the same call sites.
- You are in an async codebase and want context that follows tasks via `contextvars`.
- You want to keep stdlib `logging` for routing but standardize the *format*.

**Avoid when:**
- You have a small script and stdlib `logging` (or `print`) is enough — the configuration overhead is not worth it.
- You want a zero-configuration drop-in with batteries included — loguru is closer to that shape.
- Your only requirement is "make stdlib logging emit JSON" — a JSON formatter is less to learn.

## Alternatives

- Delgan/loguru — use instead when you want an ergonomic, near-zero-config drop-in and don't need explicit processor pipelines or dict-first semantics.
- python/cpython (stdlib `logging`) — use instead when you want zero dependencies and don't need structured context or format flexibility.
- madzak/python-json-logger — use instead when your only goal is JSON output from the standard library's `logging`.
- open-telemetry/opentelemetry-python — use instead when logs must be correlated with traces and metrics in a single telemetry pipeline.
- getsentry/sentry-python — use alongside/instead when the real need is error aggregation and alerting rather than general logging.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2013-09 | Initial release; bound-logger idea credited to Jean-Paul Calderone and David Reid[^1]. |
| 16.0.0 | 2016-01 | Switched to CalVer (`YY.MINOR.MICRO`) versioning. |
| 20.1.0 | 2020-05 | Dropped Python 2; added `contextvars` support and async logging methods (`ainfo`, etc.)[^5]. |
| 21.1.0 | 2021-05 | `ProcessorFormatter` / stdlib-interop improvements; typing refinements. |
| 22.1.0 | 2022-06 | Native async and callsite-parameter processor additions. |
| 24.1.0 | 2024-01 | Continued CalVer line; recent Python version support, renderer/exception improvements. |
| 25.x | 2025 | Current CalVer line at time of writing. |

## References

[^1]: structlog README and Credits — "successfully used in production at every scale since 2013"; bound-logger idea credited to Jean-Paul Calderone and David Reid. https://github.com/hynek/structlog
[^2]: structlog docs, "Loggers" — lazy `get_logger()` proxy and `cache_logger_on_first_use`. https://www.structlog.org/en/stable/loggers.html
[^3]: structlog docs, "Standard Library Logging" — `ProcessorFormatter` and `foreign_pre_chain`. https://www.structlog.org/en/stable/standard-library.html
[^4]: structlog docs, "Async" — `await log.ainfo(...)` runs the chain in an executor. https://www.structlog.org/en/stable/async.html
[^5]: structlog CHANGELOG — Python 2 removal, `contextvars`, and async methods in the 20.x series. https://github.com/hynek/structlog/blob/main/CHANGELOG.md

## Tags

python, logging, structured-logging, observability, json-logging, contextvars, async, library, stdlib-logging, logfmt
