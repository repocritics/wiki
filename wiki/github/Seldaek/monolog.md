# Seldaek/monolog

> The de facto logging library for PHP — a PSR-3 logger built around a stack of composable handlers, formatters, and processors.

[GitHub repo](https://github.com/Seldaek/monolog) ·
[Official website](https://seldaek.github.io/monolog/) ·
[License: MIT](https://github.com/Seldaek/monolog/blob/main/LICENSE)

## Overview

Monolog is a logging library for PHP written by Jordi Boggiano (Seldaek), first
released in 2011[^1]. It predates PSR-3 and heavily influenced that standard;
today it implements the `Psr\Log\LoggerInterface` and is the logger shipped or
recommended by nearly every major PHP framework — Symfony, Laravel, and Lumen
wire it in out of the box[^2]. In practical terms, "logging in PHP" and "Monolog"
are close to synonymous, and it is one of the most-installed packages on
Packagist.

The design is deliberately small at the core and extensible at the edges. A
`Logger` is little more than a named channel holding an ordered stack of
handlers; everything interesting — where logs go, how they are formatted, what
metadata is attached — lives in pluggable handlers, formatters, and processors.
This is the library's defining tradeoff: the flexibility that makes Monolog fit
files, syslog, Slack, Elasticsearch, Sentry, and dozens of other targets also
means the routing behavior (handler order, level thresholds, bubbling) is
implicit in how you assemble the stack, and misassembly is the most common
source of "my logs aren't showing up" bugs.

Monolog is also notable for its version discipline. The 1.x line ran for years
on old PHP; 2.x modernized the type surface; 3.x (2022) rewrote the record model
around PHP 8.1 enums and a value object[^3]. Each major bump is a real migration,
not a cosmetic one.

## Getting Started

```bash
composer require monolog/monolog
```

```php
<?php

use Monolog\Level;
use Monolog\Logger;
use Monolog\Handler\StreamHandler;

$log = new Logger('app');
$log->pushHandler(new StreamHandler(__DIR__ . '/app.log', Level::Warning));

$log->warning('Low disk space', ['free_mb' => 512]);
$log->error('Payment failed', ['order_id' => 42]);
```

The second argument to `StreamHandler` is the minimum level: records below
`Warning` are ignored by that handler. The context array (`['order_id' => 42]`)
is carried through with the record and rendered by the formatter — it is not
interpolated into the message string unless you add `PsrLogMessageProcessor`.

## Architecture / How It Works

Every log call produces a `LogRecord` carrying: a `Level`, the message string,
a `context` array (per-call data), an `extra` array (data injected by
processors), the channel name, and a `DateTimeImmutable`. In Monolog 3.x
`LogRecord` is an immutable value object and `Level` is a native enum; in 1.x/2.x
the record was a plain array and levels were integer constants — the single
biggest source of breakage when upgrading[^3].

Three extension points shape a record's fate:

1. **Handlers** — decide *where* a record goes and whether it stops there. A
   Logger holds a stack of them. Handlers are invoked in **LIFO order** (the last
   one pushed runs first). Each handler has a minimum level and a `$bubble` flag:
   when `bubble` is `false` and the handler handles a record, propagation to
   handlers lower in the stack stops. This bubbling model is powerful and the
   most frequently misunderstood part of the library.
2. **Formatters** — decide *how* a record is serialized. `LineFormatter` (the
   default for most handlers), `JsonFormatter`, `GelfMessageFormatter`,
   `LogstashFormatter`, `ElasticsearchFormatter`, and others. One formatter per
   handler.
3. **Processors** — mutate the record before handlers see it, typically
   enriching `extra`: `WebProcessor` (request URL/IP), `IntrospectionProcessor`
   (file/line/class), `GitProcessor`, `MemoryUsageProcessor`,
   `PsrLogMessageProcessor` (interpolates `{placeholders}`). Processors can be
   attached to a Logger or to an individual handler.

The genuinely clever handlers are the composite/meta ones.
`FingersCrossedHandler` buffers all records silently in memory until one crosses
an activation level (say `Error`), then flushes the entire buffer to a wrapped
handler — so you get verbose debug context only for requests that actually
failed. `BufferHandler`, `GroupHandler`, `WhatFailureGroupHandler` (isolates a
failing sub-handler), `DeduplicationHandler`, `SamplingHandler`, and
`FilterHandler` compose similarly. Understanding that handlers wrap other
handlers is the key mental model.

## Production Notes

- **Handler order is LIFO, and silent when wrong.** A missing log line is almost
  always a level threshold set too high or an earlier handler with `bubble=false`
  swallowing the record. There is no error; the record simply goes nowhere.

- **`StreamHandler` does not lock by default.** Concurrent processes writing the
  same file can interleave lines. Pass `useLocking: true` to enable `flock`, at a
  throughput cost. For high-volume production, log to `stdout`/`stderr` (or
  syslog) and let the platform aggregate, rather than writing shared files.

- **`RotatingFileHandler` rotates by date, not by size.** It relies on a date in
  the filename and prunes by `maxFiles` count. If you need size-based rotation,
  use OS-level `logrotate` against a `StreamHandler` and signal-reopen, or a
  dedicated handler.

- **`FingersCrossedHandler` holds records in memory.** In a long-running worker
  (queue consumer, Swoole/RoadRunner, daemon) the buffer accumulates across the
  process lifetime unless you call `reset()` between units of work. Monolog
  exposes `Logger::reset()` and the `ResettableInterface` precisely for this;
  forgetting it leaks memory and cross-contaminates request context.

- **Context is not interpolated by default.** `$log->info('User {id}', ['id' =>
  5])` logs the literal `{id}` unless `PsrLogMessageProcessor` is registered.
  Framework integrations usually add it; hand-rolled setups often do not.

- **Third-party service handlers add latency and failure modes.** Sending logs
  synchronously to Slack, Elasticsearch, or an HTTP endpoint blocks the request
  and can throw. Wrap them in `WhatFailureGroupHandler` (never re-throws) and/or
  `BufferHandler`, or push to a local sink and ship asynchronously.

- **Upgrade cost is real.** 2.x → 3.x changes level constants to the `Level` enum
  and the record array to `LogRecord`; custom handlers, formatters, and
  processors that typed against arrays or `int` levels must be rewritten. 1.x is
  effectively end-of-life and receives only critical fixes[^2].

## When to Use / When Not

**Use when:**
- You are on any modern PHP stack and need application logging — this is the
  default and safe choice.
- You need to fan logs out to multiple destinations with per-destination levels
  and formats.
- You want conditional verbosity (log everything only when something breaks) via
  `FingersCrossedHandler`.

**Avoid / reconsider when:**
- You only need the interface to type-hint against — depend on `psr/log`, not
  Monolog.
- You want a unified traces + metrics + logs pipeline — reach for OpenTelemetry
  PHP and bridge Monolog into it rather than treating Monolog as the whole story.
- You are in an extremely hot path where even in-process record allocation
  matters — gate log calls behind level checks; Monolog constructs records
  eagerly.

## Alternatives

- php-fig/log — the PSR-3 interface only, no implementation; depend on it when you
  just need the contract and let the app pick the logger.
- open-telemetry/opentelemetry-php — use when logs must join traces and metrics in
  one observability pipeline rather than living alone.
- laminas/laminas-log — use inside Laminas/Mezzio applications where its component
  wiring is already present.
- apix/log — use when you want a lightweight PSR-3 logger with a smaller surface
  than Monolog's handler zoo.
- symfony/monolog-bundle — not an alternative but the Symfony integration; use it
  to configure Monolog declaratively instead of building the stack by hand.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2011-09 | Initial release; handler/formatter/processor model, array records[^1]. |
| 1.11.0 | 2015 | Public APIs accept PSR-3 string levels. |
| 2.0.0 | 2019-11 | PHP 7.2+, typed properties, `DateTimeImmutable`, stricter types[^2]. |
| 3.0.0 | 2022-05 | PHP 8.1+, `Level` enum, `LogRecord` value object[^3]. |

## References

[^1]: Monolog repository and history, Jordi Boggiano (Seldaek). https://github.com/Seldaek/monolog
[^2]: Monolog README — framework integrations and version support (Symfony, Laravel, Lumen ship it by default; 1.x limited support). https://github.com/Seldaek/monolog/blob/main/README.md
[^3]: Monolog 3.0 upgrade notes — `Level` enum and `LogRecord` object. https://github.com/Seldaek/monolog/blob/main/UPGRADE.md

## Tags

php, logging, logger, psr-3, monolog, observability, library, backend, handlers, structured-logging
