# pinojs/pino

> A low-overhead Node.js JSON logger that keeps the hot path cheap and pushes formatting and shipping off to a separate thread.

[GitHub repo](https://github.com/pinojs/pino) ·
[Official website](http://getpino.io) ·
[License: MIT](https://github.com/pinojs/pino/blob/main/LICENSE)

## Overview

Pino is a structured logging library for Node.js that emits newline-delimited JSON (NDJSON) to a file descriptor — stdout by default[^1]. It was created inside nearForm (Matteo Collina, David Mark Clements, and others) and is now sponsored by Platformatic[^2]. Its guiding thesis is that the logging call on the application's critical path should do as little work as possible: pino serializes a shallow object and writes bytes, and defers everything expensive — pretty-printing, filtering, shipping to Elasticsearch or a syslog collector — to a downstream process or worker thread.

The design follows the twelve-factor convention: an application logs an event stream to stdout and lets the surrounding infrastructure (systemd, Docker, a log agent, a platform) decide where it goes. This makes pino a natural fit for containerized and serverless deployments, and the reason it is the default logger in Fastify (same primary author)[^3]. The tradeoff is that pino is opinionated about *not* doing things: it will not, in its fast path, colorize output, rotate files, or format timestamps for humans. Those are deliberately someone else's job, which surprises developers arriving from Winston or `console.log`.

The other defining choice is **numeric log levels**. By default pino writes `"level":30` rather than `"level":"info"`, because comparing integers is cheaper than strings. This is invisible once logs reach a system that understands the mapping, and a persistent source of confusion for anyone reading raw output for the first time.

## Getting Started

```bash
npm install pino
# pretty output for local dev is a separate module:
npm install --save-dev pino-pretty
```

```js
const pino = require('pino')

// Default: NDJSON to stdout, level "info" and above.
const logger = pino()
logger.info('hello world')
logger.info({ userId: 42, route: '/checkout' }, 'request handled')

// Child loggers bind fields onto every subsequent line — cheap, common pattern.
const reqLog = logger.child({ requestId: 'abc-123' })
reqLog.warn('slow downstream call')

// Redaction is configured up front, by object path.
const secure = pino({
  redact: ['req.headers.authorization', 'password'],
})
```

Output (one JSON object per line):

```
{"level":30,"time":1690000000000,"pid":657,"hostname":"host","msg":"hello world"}
```

For local development, pipe through the CLI rather than formatting in-process: `node app.js | pino-pretty`.

## Architecture / How It Works

At the center is a logger object whose methods (`info`, `error`, …) are generated per configured level. On a call, pino builds a shallow JSON string from the bound child bindings, the passed object, and the message, then hands the buffer to **sonic-boom**, a fast append-only stream writer that batches `write(2)` syscalls[^4]. Child loggers do not copy state; they prepend a pre-serialized bindings fragment, so `logger.child({...})` is near-free and encouraged per request.

Serialization avoids `JSON.stringify` on the whole payload. Standard **serializers** (`err`, `req`, `res`) convert well-known shapes; custom serializers handle the rest, and circular references are handled by a safe stringifier. **Redaction** is compiled ahead of time from path expressions via `fast-redact` into a specialized function rather than walking the object at log time.

The most consequential architectural piece is **transports**. Since v7, log processing runs in a Worker thread launched by `pino.transport(...)`; the main thread only writes to a message port, and the worker does the CPU-bound reformatting and network I/O[^5]. This keeps a single-threaded event loop from stalling on log shipping. Before v7, the same separation was achieved by piping stdout to a separate OS process (`node app | pino-elasticsearch`), which still works and is often simpler to reason about in production.

`pino/file` and the default destination can run in **asynchronous mode** (`sync: false`), buffering writes in memory and flushing periodically. This is where most of pino's throughput advantage comes from, and also its sharpest edge (see below).

## Production Notes

- **Numeric levels reach your log store.** Downstream tooling must map `30 → info`. If you need labels in the raw output, set `formatters: { level: (label) => ({ level: label }) }` — but note that custom `formatters` are not supported inside worker-thread transports in all configurations, so decide early where the mapping happens.
- **Async logging can drop the tail on a crash.** In `sync: false` mode, buffered lines are lost if the process exits abnormally before a flush. Install `logger.flush()` on graceful shutdown and consider `pino.destination({ sync: true })` for the last-resort fatal path. There is no free lunch: synchronous logging is safe but costs event-loop time.
- **Never run pino-pretty in production.** It is synchronous and comparatively slow; it exists for human eyes during development. Emit JSON in prod and prettify at read time.
- **Transport errors can be quiet.** A misconfigured worker transport (bad target, network failure) may fail without crashing the app. Attach an `error` handler to the transport and monitor that the worker is alive; a common outage mode is "app is fine, logs silently stopped."
- **Redaction is path-based, not content-based.** `fast-redact` matches known paths (with limited wildcards). It will not catch a secret that lands under an unexpected key or inside a stringified blob. Treat it as defense-in-depth, not a guarantee.
- **Worker transports add process complexity.** Under some bundlers, PM2 cluster mode, or restricted runtimes, spawning a Worker for the transport is fragile. The pipe-to-separate-process model (v6-style) is often more robust in those environments.
- **Deep/large objects still cost.** The "fast" claim applies to the shallow common case; logging big nested payloads on every request will show up in profiles regardless of the logger.

## When to Use / When Not

**Use when:**
- You want structured JSON logs with minimal event-loop overhead.
- You deploy to containers/serverless where "log to stdout, ship elsewhere" is the norm.
- You use Fastify, or otherwise want the ecosystem default with `pino-http` request logging.
- You need per-request context via cheap child loggers.

**Avoid when:**
- You need rich in-process formatting, many built-in transports, or human-readable files without a second process — Winston fits that shape better.
- You are writing a CLI where pretty, colorized, human-first output is the primary product — a DX-focused logger is a better default.
- Your team will read raw log files by eye and cannot tolerate numeric levels or a separate pretty step.

## Alternatives

- winstonjs/winston — more configurable, many first-party transports and formats done in-process; use when flexibility matters more than hot-path overhead.
- trentm/node-bunyan — the earlier JSON-logger design pino descends from; use only for legacy code, as it is effectively unmaintained.
- unjs/consola — pretty, developer-friendly console output; use for CLIs and build tools rather than high-throughput services.
- visionmedia/debug — namespaced on/off debug logging for libraries; use for opt-in diagnostic tracing, not structured application logs.
- log4js-node — appender/category model familiar from Java's log4j; use when you specifically want that configuration style.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016 | Initial release; NDJSON to stdout, numeric levels[^1]. |
| 6.x | 2020 | Widely deployed line; separate maintained `v6.x` branch[^6]. |
| 7.0 | 2021-10 | Worker-thread transports via `pino.transport`[^5]. |
| 8.0 | 2022-06 | Dropped older Node versions; transport model matured[^6]. |
| 9.0 | 2024-04 | Further Node baseline bump; ongoing maintenance[^6]. |

## References

[^1]: pino README and documentation. https://github.com/pinojs/pino
[^2]: pino Acknowledgments — sponsored by nearForm, now Platformatic. https://github.com/pinojs/pino#acknowledgments
[^3]: Fastify logging documentation — pino is the default logger. https://fastify.dev/docs/latest/Reference/Logging/
[^4]: sonic-boom — fast append-only stream writer used by pino. https://github.com/pinojs/sonic-boom
[^5]: pino Transports documentation (worker-thread model). https://github.com/pinojs/pino/blob/main/docs/transports.md
[^6]: pino releases. https://github.com/pinojs/pino/releases

## Tags

javascript, nodejs, logging, json-logger, structured-logging, observability, ndjson, fastify, performance, backend
