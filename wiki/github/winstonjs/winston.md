# winstonjs/winston

> A logger for just about everything — the multi-transport Node.js logging library that treats every logger and destination as an object-mode stream.

[GitHub repo](https://github.com/winstonjs/winston) ·
[npm package](https://www.npmjs.com/package/winston) ·
[License: MIT](https://github.com/winstonjs/winston/blob/master/LICENSE)

## Overview

Winston is one of the oldest and most widely deployed Node.js logging libraries, first published in 2010[^1]. Its organizing idea is the **transport**: a logger holds an array of transports (Console, File, Http, or community-authored destinations), each with its own level and format, and a single `logger.info(...)` call fans out to all of them. This decoupling — formatting separated from levels separated from where logs actually land — is the reason it survived long enough to become a default choice for application logging.

The design tension is configurability versus overhead. Winston is deliberately flexible: custom levels, composable formats, per-transport filtering, exception/rejection handling, querying and streaming of stored logs. That surface area makes it heavier and slower than newer JSON-first loggers. Teams that value structured, high-throughput logging with minimal per-line cost increasingly reach for pino instead; teams that value one library that can log to a file, a console, and a remote service with different formats each tend to stay on winston.

Winston 3.x (2018) is a near-total rewrite of the 2.x line[^2]. It rebased the whole library on Node.js streams and extracted the formatting engine into a separate package, [`logform`](https://github.com/winstonjs/logform). Most tutorials and Stack Overflow answers older than 2018 describe the 2.x API and do not apply.

## Getting Started

```bash
npm install winston
```

```js
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: { service: 'user-service' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// Console only outside production, with a human-readable format
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple(),
  }));
}

logger.info('server started', { port: 3000 });
```

## Architecture / How It Works

Winston is built on Node.js **object-mode streams**[^3]. Both a `Logger` and a `Transport` are stream instances; logging pushes an `info` object (`{ level, message, ...meta }`) through the pipeline. This is why transports can apply backpressure and why the library integrates with anything that speaks streams.

The pipeline has three separable concerns:

- **Levels** — an integer severity map, ascending in importance as the number decreases, per RFC5424. The default is the `npm` set (`error:0 … silly:6`); `syslog` and `cli` sets ship built-in, and you can supply your own. A transport's `level` is the *maximum* verbosity it will record.
- **Formats** — live in `logform`, not winston itself, so custom transports can bundle a default format. A format is an object with a single `transform(info, opts)` method that mutates and returns `info`, or returns a falsey value to drop the message entirely. `format.combine(...)` chains them and short-circuits on the first drop. "Finalizing" formats (`json`, `simple`, `printf`, `logstash`, `prettyPrint`) write the rendered string onto `info[Symbol.for('message')]`.
- **Transports** — the base class lives in the separate `winston-transport` package. Built-ins are Console, File, Http, and Stream; the ecosystem adds MongoDB, Elasticsearch, daily-rotate-file, Loggly, CloudWatch, and dozens more.

Internal state travels on `Symbol` properties (`LEVEL`, `MESSAGE`, `SPLAT`) exported from the `triple-beam` package, so `logform`, `winston-transport`, and winston all share one Symbol identity. String interpolation (`%s`, `%d`) is **not on by default** — it requires `format.splat()` in the format chain, a frequent source of confusion for users porting `printf`-style code.

The practical consequence of the stream architecture: winston, logform, winston-transport, and triple-beam are four coupled packages that must stay version-compatible. A mismatch (often dragged in transitively by a community transport) produces subtle format or Symbol bugs rather than a clean error.

## Production Notes

**The default logger has no transports.** `require('winston').info(...)` works, but with zero transports attached winston buffers messages, which the README explicitly warns can cause a memory leak under load[^1]. Always create a logger with explicit transports for anything real.

**Logs are not flushed synchronously before exit.** Because transports are async streams, calling `process.exit()` right after a `logger.error(...)` can truncate or lose the last lines — especially with the File transport. Wait for the transport's `'finish'`/`'close'` event (or the logger's callback) before exiting. This is the single most common "my last log vanished" bug.

**File transport is not a rotation solution.** The built-in File transport supports `maxsize` and `maxFiles` rolling but not time-based rotation. Production setups almost always add `winston-daily-rotate-file`, or log to stdout and let the platform (Docker, systemd, a log shipper) handle rotation.

**Throughput.** Winston's flexibility costs CPU per line — format composition, object cloning, and stream plumbing add up. Benchmarks consistently show pino several times faster at high volume because pino defers formatting and serializes to newline-delimited JSON directly. If logging is on a hot path, measure before committing to winston.

**`logger.child()` caveat.** Child loggers (for attaching request-scoped metadata) are documented in the README as likely to misbehave if you also subclass `Logger`, because of how `this` binds internally[^1]. Prefer composition over extending the `Logger` class.

**Exception handling** is opt-in via `exceptionHandlers`/`rejectionHandlers` transports, and `exitOnError` controls whether a handled uncaught exception still calls `process.exit`. Enabling these means winston becomes part of your crash path — test that the handler transport actually flushes.

**Version skew.** Because formatting lives in `logform` and the base transport in `winston-transport`, upgrading winston without upgrading these (or a lagging community transport pinning old versions) is the usual root cause of "format is being ignored" reports. Check the resolved versions of all four packages when formats behave unexpectedly.

## When to Use / When Not

**Use when:**
- You need multiple destinations with different levels and formats from one logger (e.g. JSON to a file, colorized text to console, errors to a remote service).
- You want custom levels, composable formats, or built-in exception/rejection capture.
- You rely on the large ecosystem of community transports (Elasticsearch, CloudWatch, MongoDB, Datadog, etc.).
- Logging volume is moderate and developer ergonomics matter more than raw throughput.

**Avoid when:**
- Logging is on a latency- or throughput-critical path — pino is meaningfully faster.
- You want structured JSON logs with minimal ceremony and don't need multi-transport routing.
- You're on the edge/serverless runtimes where the streams/`fs` model is awkward; a stdout-only JSON logger fits better.
- You only need namespaced dev debugging — `debug` is lighter.

## Alternatives

- pinojs/pino — JSON-first, low-overhead logger; use instead when throughput and latency are the priority and you're happy with newline-delimited JSON to stdout.
- trentm/node-bunyan — structured JSON logging with a companion CLI pretty-printer; use when you want bunyan's log-viewing tooling, accepting slower maintenance cadence.
- log4js-node — log4j-style appenders and categories; use when you want a configuration model familiar from the Java logging world.
- debug — namespaced, environment-toggled debug output; use instead when you only need conditional developer logging, not durable structured logs.
- winstonjs/logform — winston's own formatting engine; not a replacement but usable standalone if you want winston's formats in a custom logging stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2010-12 | Initial release by Charlie Robbins (Nodejitsu)[^1]. |
| 1.0.0 | 2015-04 | First 1.0 API stabilization. |
| 2.0.0 | 2015-10 | Reworked transports and configuration. |
| 3.0.0 | 2018-06 | Full rewrite on Node.js streams; formats extracted to `logform`; Symbols moved to `triple-beam`[^2]. |
| 3.x | ongoing | Active 3.x line; incremental transport/format fixes on `master`. |

## References

[^1]: winston README and package history, winstonjs/winston. https://github.com/winstonjs/winston
[^2]: winston 3.0 Upgrade Guide (UPGRADE-3.0.md). https://github.com/winstonjs/winston/blob/master/UPGRADE-3.0.md
[^3]: Node.js Stream documentation, object mode. https://nodejs.org/api/stream.html#object-mode

## Tags

javascript, nodejs, logging, logger, transports, observability, structured-logging, streams, backend, npm-package
