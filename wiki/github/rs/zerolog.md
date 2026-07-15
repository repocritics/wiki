# rs/zerolog

> A zero-allocation JSON logger for Go, built around a typed method-chaining API.

[GitHub repo](https://github.com/rs/zerolog) ·
[GoDoc](https://pkg.go.dev/github.com/rs/zerolog) ·
[License: MIT](https://github.com/rs/zerolog/blob/master/LICENSE)

## Overview

zerolog is a structured logging library for Go written by Olivier Poitrey (`rs`), first tagged in 2017[^1]. Its defining idea is to build each log line through a chain of strongly-typed methods — `log.Info().Str("k", v).Int("n", 1).Msg("...")` — that append directly into a byte buffer without reflection and, in the common path, without heap allocations. It explicitly follows the approach Uber's zap pioneered, trading a conventional `Printf`-style API for a builder that the compiler can keep on the stack[^2].

The library is deliberately narrow: it does structured JSON (and CBOR) well and treats everything else as secondary. Human-readable console output exists via `ConsoleWriter`, but it is reflection-based and allocation-heavy, documented as development-only. There is no built-in log rotation, no config file, no pluggable formatter zoo — output is JSON, levels are fixed, and customization happens through global variables and hooks. That focus is why it is fast, and also why teams that want a general-purpose logging framework sometimes find it spartan.

The central tension in zerolog is that its performance comes from a mutable, pooled `Event` object and a chain you must terminate correctly. The API is ergonomic until you hit its two unforgiving rules (finalize every chain; never reuse a dispatched event), at which point the same design that avoids allocations also avoids compile-time safety.

## Getting Started

```bash
go get -u github.com/rs/zerolog/log
```

```go
package main

import (
	"os"

	"github.com/rs/zerolog"
)

func main() {
	logger := zerolog.New(os.Stdout).
		With().
		Timestamp().
		Str("service", "billing").
		Logger()

	logger.Info().
		Str("user", "u_123").
		Int("cents", 4200).
		Msg("charge captured")

	// {"level":"info","service":"billing","time":"2026-...","user":"u_123","cents":4200,"message":"charge captured"}
}
```

The `github.com/rs/zerolog/log` subpackage ships a package-level global logger (`log.Info()...`) writing to `os.Stderr`, convenient for applications; construct your own `zerolog.Logger` when you need distinct outputs or context.

## Architecture / How It Works

A `Logger` is a small value holding an `io.Writer`, a minimum level, and a pre-serialized context buffer. Calling a level method (`Info()`, `Error()`, …) pulls an `*Event` from a `sync.Pool`. Each field method (`Str`, `Int`, `Dur`, `Interface`, …) appends the key and encoded value straight into the event's `[]byte`. `Msg()` / `Send()` writes the buffer to the output and returns the event to the pool. Because encoding is hand-written per type rather than via `reflect`, and the event is pooled, the hot path allocates nothing once the pool is warm.

Levels run from `TraceLevel` (-1) through `DebugLevel` (0) up to `PanicLevel` (5)[^3]. When an event's level is below the logger's threshold (or the global level set by `SetGlobalLevel`), the level method returns a disabled event whose field calls are cheap no-ops — the reason you can leave debug logging in hot code with little cost, provided you avoid `Msgf` (its formatting allocates even when disabled).

Contextual logging has two shapes. `With().…().Logger()` returns a *new* logger whose context fields are serialized once and prepended to every subsequent event (sub-loggers). Per-event fields are added on the chain itself. The encoder does not deduplicate keys, so a field set on both the context and the event appears twice in the output JSON.

Notable surrounding packages: `hlog` integrates with `net/http` handlers (request-id, access logging); `pkgerrors` marshals stack traces from `github.com/pkg/errors`; CBOR binary output is enabled with the `binary_log` build tag; and a `slog.Handler` adapter lets zerolog back Go's standard `log/slog` API, added in the 2023 release line[^4]. A `diode.Writer` (via `code.cloudfoundry.org/go-diodes`) provides a lock-free, dropping, non-blocking writer for cases where a slow sink must never back-pressure producers.

## Production Notes

- **Unterminated chains silently drop.** `log.Info().Str("k","v")` without a trailing `Msg`/`Send`/`Msgf` produces no output and no compile error. This is the most common zerolog bug. `go vet` does not catch it by default; the community linter `zerologlint` does and is worth adding to CI.
- **Never reuse a dispatched event.** The `*Event` is returned to the pool by `Msg()`. Holding onto it (e.g. `e := log.Info(); e.Msg("a"); e.Msg("b")`) reuses freed memory and causes races and corrupted output. Treat each chain as single-use.
- **Global settings are init-time only.** `zerolog.TimeFieldFormat`, `TimestampFieldName`, `ErrorStackMarshaler`, and similar package variables are plain globals with no synchronization. Set them once at startup; mutating them while other goroutines log is a data race.
- **`ConsoleWriter` is not for production.** It re-parses each JSON line and uses reflection to colorize; it allocates heavily and is single-threaded-slow. Keep JSON in prod and pretty-print at the consumer (e.g. pipe through a viewer) instead.
- **`Fatal()` calls `os.Exit(1)`**, bypassing deferred functions; `Panic()` panics after logging. Neither is appropriate deep inside request handlers.
- **Duplicate keys are legal output.** Because keys are not deduplicated, mixing context and event fields with the same name yields JSON with repeated keys; downstream parsers vary in which value wins. Establish field-name conventions.
- **Sampling is per-logger and probabilistic.** `BasicSampler`, `BurstSampler`, and `LevelSampler` reduce volume but mean you cannot assume every event was written — size alerting accordingly, or exempt error levels from sampling.
- **`Caller()` has real cost.** It calls `runtime.Caller` per event; enable it selectively rather than globally in high-throughput paths.

## When to Use / When Not

**Use when:**
- You want machine-readable JSON logs and care about allocation/GC pressure under load.
- You are shipping a service whose logs feed a collector (Loki, Elasticsearch, CloudWatch) rather than human eyes.
- You want a small, stable dependency with a typed field API and `net/http` / `slog` / CBOR integration available.

**Avoid when:**
- You primarily want pretty, human-first console logs — `ConsoleWriter`'s cost undercuts the library's reason to exist; a console-oriented logger fits better.
- You want compile-time guarantees that a log statement is complete — the chain-must-be-terminated footgun is inherent to the design.
- You are standardizing on Go's stdlib `log/slog` and don't need zerolog's throughput; the standard handler may suffice (zerolog can also sit behind slog if you change your mind).

## Alternatives

- uber-go/zap — the library zerolog is modeled on; more configuration surface and a `SugaredLogger` escape hatch, at the cost of a larger API.
- golang/go `log/slog` — the standard-library structured logger since Go 1.21; use it when a stdlib dependency and portability matter more than peak throughput.
- sirupsen/logrus — older, reflection-based, widely adopted but in maintenance mode; choose it only for legacy compatibility, not performance.
- phuslu/log — a zerolog-style API chasing even lower overhead; consider when micro-benchmarked allocation counts are the deciding factor.
- charmbracelet/log — console-first, styled output; the right pick when human readability, not JSON pipelines, is the goal.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2017-06-08 | First tagged release; chaining API and JSON encoder[^1]. |
| v1.20.0 | 2020-08-06 | Continued expansion of field types, hooks, and sampling. |
| v1.29.0 | 2023-01-25 | `context.Context` carried into events for hook-based tracing. |
| v1.30.0 | 2023-07-29 | `log/slog` handler adapter added around this line[^4]. |
| v1.33.0 | 2024-05-04 | Ongoing encoder and `ConsoleWriter` formatting refinements. |
| v1.35.1 | 2026-04-20 | Latest tag at time of writing. |

## References

[^1]: rs/zerolog release tags. https://github.com/rs/zerolog/tags
[^2]: Project README, "Uber's zap library pioneered this approach." https://github.com/rs/zerolog#readme
[^3]: GoDoc — level constants (`TraceLevel` … `PanicLevel`). https://pkg.go.dev/github.com/rs/zerolog#Level
[^4]: zerolog `slog` handler documentation. https://pkg.go.dev/github.com/rs/zerolog#SlogHandler

## Tags

go, golang, logging, structured-logging, json, cbor, zero-allocation, observability, slog, library
