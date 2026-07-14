# uber-go/zap

> Uber's zero-allocation structured logging library for Go, built around typed
> fields instead of reflection.

[GitHub repo](https://github.com/uber-go/zap) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/go.uber.org/zap) ·
[License: MIT](https://github.com/uber-go/zap/blob/master/LICENSE)

## Overview

zap is a structured, leveled logging library for Go, open-sourced by Uber in
2016[^1]. Its defining bet is that the cost of logging in a hot path is
dominated by reflection and allocation — the same overhead you pay when you
hand `interface{}` values to `encoding/json` or `fmt.Fprintf`. zap sidesteps
this with strongly typed field constructors (`zap.String`, `zap.Int`,
`zap.Duration`, …) and a reflection-free, allocation-conscious JSON encoder, so
that a log call in steady state can allocate zero objects[^2].

The library exposes this tradeoff directly through two APIs. The base `Logger`
takes typed `Field` values and is the fast path. The `SugaredLogger`, obtained
via `logger.Sugar()`, adds loosely typed key-value and `printf`-style methods
(`Infow`, `Infof`) at the cost of some allocations and speed. This "choose your
poison" design is the whole point of zap: you opt into ceremony only where the
allocation count matters, and use the ergonomic API everywhere else.

The project has been API-stable since its 1.0 release and commits to no
breaking changes within the 1.x line[^3]. That stability is a feature for
long-lived services but also means zap predates — and does not natively mirror
— Go's own `log/slog`, which arrived in the standard library in Go 1.21.
Interop is provided through a separate `zapslog` handler rather than baked into
the core[^4].

## Getting Started

```bash
go get -u go.uber.org/zap
```

```go
package main

import (
	"time"

	"go.uber.org/zap"
)

func main() {
	logger, _ := zap.NewProduction()
	defer logger.Sync() // flush buffered entries before exit

	// Fast path: typed fields, zero-reflection.
	logger.Info("failed to fetch URL",
		zap.String("url", "https://example.com"),
		zap.Int("attempt", 3),
		zap.Duration("backoff", time.Second),
	)

	// Ergonomic path: loosely typed, slower.
	sugar := logger.Sugar()
	sugar.Infow("failed to fetch URL", "attempt", 3)
	sugar.Infof("failed to fetch URL: %s", "https://example.com")
}
```

`NewProduction` emits JSON at `Info` level with sampling enabled;
`NewDevelopment` emits human-readable console output at `Debug` level. For real
control, build a `zap.Config` or assemble a `zapcore.Core` directly.

## Architecture / How It Works

zap is two layers. The user-facing `zap` package (`Logger`, `SugaredLogger`,
the `Field` constructors) sits on top of `zapcore`, the extension layer that
does the actual work[^5]. Understanding `zapcore` is the difference between
using zap and configuring it.

- **`zapcore.Core`** is the unit that decides whether an entry is logged and
  where it goes. `Core` bundles three things: a `LevelEnabler` (is this level
  on?), an `Encoder` (how to serialize), and a `WriteSyncer` (where bytes go).
- **`Encoder`** — `NewJSONEncoder` and `NewConsoleEncoder` are built in. The
  JSON encoder is the reflection-free component that makes the benchmarks work;
  typed `Field`s call directly into it with no `interface{}` round-trip.
- **`WriteSyncer`** — an `io.Writer` plus `Sync()`. zap buffers writes, so
  `Sync()` is what flushes them. `zapcore.AddSync` and `zapcore.Lock` wrap a
  plain writer into a concurrency-safe syncer.
- **`Field`** — a tagged union (`zapcore.Field`) carrying a type tag and the
  value in a non-boxed slot where possible (integers in an `int64`, etc.), so
  common types never touch reflection. `zap.Any` is the escape hatch that
  *does* fall back to reflection.

Composition happens at the `Core` level. `zapcore.NewTee` fans one logger out
to multiple cores (e.g. JSON to a file and console to stderr at different
levels). `logger.With(fields...)` returns a child logger that pre-encodes those
fields once, so repeated logs with shared context pay the encoding cost a
single time — this is why the "with 10 fields of context" benchmark allocates
zero.

Sampling is also a `Core` (`zapcore.NewSamplerWithOptions`). The production
preset wraps its core in a sampler that, per level per second, logs the first N
entries then every Mth thereafter — trading completeness for bounded output
under log storms.

## Production Notes

**`Sync()` errors are a known footgun.** You must call `logger.Sync()` before
exit to flush buffered entries, but syncing when the sink is a terminal or
`/dev/stderr` returns an error on some platforms — notably "inappropriate ioctl
for device" on macOS and "invalid argument" on Linux[^6]. The common
`defer logger.Sync()` idiom silently swallows this; ignoring the error there is
usually correct, but do not treat a `Sync` error as fatal on stdio sinks.

**The global logger is a no-op until you replace it.** `zap.L()` and `zap.S()`
return a no-op logger by default; nothing is emitted until you call
`zap.ReplaceGlobals(logger)`. Libraries that log via the globals will appear
silent in a program that never wired them up.

**`DPanic` behaves differently by build.** At `DPanic` level zap panics when the
logger is in development mode and merely logs at panic level in production. Code
that relies on the panic for control flow will behave differently depending on
which constructor built the logger.

**SugaredLogger is not free.** It is 4–10x faster than reflection-based loggers
but still allocates relative to the typed `Logger`; the sugared path boxes
arguments into `interface{}`. Keep genuine hot paths on the typed API and
reserve `Sugar()` for cold or convenience code.

**Caller and stack skipping.** When you wrap zap in your own helper, the
reported caller line points at your wrapper unless you add
`zap.AddCallerSkip(n)`. This surfaces immediately once you centralize logging
behind a package-local function.

**Sampling can hide logs.** The production preset drops repeated entries by
design. During an incident this can make a tight error loop under-report;
disable or tune sampling in `zap.Config` when you need every line.

## When to Use / When Not

**Use when:**
- You log in hot paths and allocation count / GC pressure is a measured concern.
- You want structured JSON output with fine-grained control over encoders,
  sinks, levels, and sampling via `zapcore`.
- You need a library that has been API-stable for years and will not break on
  minor upgrades.

**Avoid when:**
- You want the leanest possible allocations and a chainable API — zerolog edges
  zap in most benchmarks.
- You want a standard-library-only dependency footprint — `log/slog` covers
  basic structured logging with nothing to vendor.
- Your logging is not on a hot path and ergonomics matter more than nanoseconds
  — the typed-field ceremony is pure cost there.

## Alternatives

- rs/zerolog — use when you want minimal or zero allocations and a fluent
  chained-call API; typically faster than zap in the same benchmarks.
- golang/go (log/slog) — use when you want the standard library's structured
  logger with no third-party dependency; zap interops via a `zapslog` handler.
- sirupsen/logrus — use when you want the most widely adopted Go logging API and
  performance is not critical; now in maintenance mode.
- apex/log — use when you want a simpler handler-based structured logger and can
  accept substantially higher per-call cost.
- uber-go/zap's own `SugaredLogger` — use within zap when you want loosely typed
  ergonomics without leaving the library.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-02 | Open-sourced by Uber[^1]. |
| 1.0.0 | 2017-08 | First stable release; 1.x API-stability commitment[^3]. |
| 1.x | 2017–2026 | Long stable line: config-driven setup, console encoder, sampling, buffered writer, hooks. |
| exp `zapslog` | 2023–2024 | `log/slog` handler shipped in the `go.uber.org/zap/exp` module after Go 1.21[^4]. |

## References

[^1]: uber-go/zap repository, created 2016-02-18. https://github.com/uber-go/zap
[^2]: zap README, "Performance" — reflection-free JSON encoder and zero-allocation base logger. https://github.com/uber-go/zap#performance
[^3]: zap README, "Development Status: Stable" — no breaking changes in the 1.x series. https://github.com/uber-go/zap#development-status-stable
[^4]: `zapslog` package — `slog.Handler` backed by zap, in the exp module. https://pkg.go.dev/go.uber.org/zap/exp/zapslog
[^5]: `zapcore` package documentation — Core, Encoder, WriteSyncer, LevelEnabler. https://pkg.go.dev/go.uber.org/zap/zapcore
[^6]: zap issue discussion on `Sync` errors for stdout/stderr sinks. https://github.com/uber-go/zap/issues/328

## Tags

go, golang, logging, structured-logging, leveled-logging, observability, performance, zero-allocation, json, backend, uber
