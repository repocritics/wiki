# sirupsen/logrus

> Structured, pluggable logging for Go — the library that popularized field-based logging in the ecosystem, now in maintenance mode.

[GitHub repo](https://github.com/sirupsen/logrus) ·
[License: MIT](https://github.com/sirupsen/logrus/blob/master/LICENSE)

## Overview

Logrus is a structured logger for Go, API-compatible with the standard
library's `log` package. First published in 2013[^1], it introduced most Go
developers to the idea of attaching typed key/value fields to log lines
(`WithFields`) rather than formatting everything into a single string. For much
of the 2010s it was the default choice for structured logging in Go services,
and its `Entry` / `Hook` / `Formatter` model shaped the APIs that came after it.

The maintainers have declared Logrus **in maintenance mode**: the project takes
security fixes, bug fixes, and performance work, but no new features aside from
interoperability with adjacent ecosystems such as Go's standard `log/slog`[^2].
The README itself points newcomers toward rs/zerolog, uber-go/zap, and apex/log,
and states there is no plan for a breaking "v2." Read this as an honest signal:
Logrus is stable and widely deployed, but it is a finished design, not an
evolving one.

The defining tension is performance versus ergonomics. Logrus's field API is
built on `map[string]interface{}`, which means every log call with fields
allocates a map and boxes values through interfaces. That was an acceptable
tradeoff in 2014; by the time zap and zerolog demonstrated near-zero-allocation
structured logging, Logrus's design could not close the gap without breaking its
API. It remains convenient and ubiquitous, but it is not the tool to reach for
on a hot path.

## Getting Started

```bash
go get github.com/sirupsen/logrus
```

```go
package main

import (
	"os"

	"github.com/sirupsen/logrus"
)

func main() {
	log := logrus.New()
	log.SetOutput(os.Stdout)
	log.SetFormatter(&logrus.JSONFormatter{}) // machine-parseable output
	log.SetLevel(logrus.InfoLevel)

	log.WithFields(logrus.Fields{
		"animal": "walrus",
		"size":   10,
	}).Info("A group of walrus emerges from the ocean")
}
```

Because the package-level API mirrors the stdlib logger, an existing codebase
can swap `import "log"` for `import log "github.com/sirupsen/logrus"` and keep
compiling, then adopt fields incrementally.

## Architecture / How It Works

Logrus is built from a small set of collaborating types:

- **`Logger`** — holds the output `io.Writer`, the active `Formatter`, the level
  threshold, registered hooks, and a mutex. `logrus.New()` creates one; a
  process-global default logger backs the package-level functions.
- **`Entry`** — a single log event plus its accumulated fields. `WithField` /
  `WithFields` return a new `Entry`, so a context entry (e.g. one carrying
  `request_id`) can be created once and reused across many log calls.
- **`Formatter`** — an interface with one method, `Format(*Entry) ([]byte,
  error)`. `TextFormatter` (logfmt-style, colorized on a TTY) is the default;
  `JSONFormatter` emits one JSON object per line. Custom formatters are trivial
  to write.
- **`Hook`** — an interface invoked on every matching entry before it is
  written, used to fan out logs to syslog, error trackers, StatsD, and similar
  sinks. Hooks run synchronously inside the write path.

The cost model follows directly from `Fields` being `map[string]interface{}`.
Each fielded call allocates the map and reflects over interface values during
formatting. Enabling `SetReportCaller(true)` adds runtime stack inspection to
attach the calling function, which the README measures at roughly 20–40%
additional overhead[^1]. Writes are serialized by the logger's mutex; hooks and
formatting all happen while that lock is held.

Logrus predates Go's standard structured logger. `log/slog` landed in the
standard library in Go 1.21 (2023)[^3] and is now the community's default for
new code; Logrus's remaining feature work is largely about coexisting with it.

## Production Notes

- **The `Sirupsen` → `sirupsen` rename.** The GitHub account was renamed to
  lowercase, and because Go import paths are case-sensitive, projects that mixed
  `github.com/Sirupsen/logrus` and `github.com/sirupsen/logrus` ended up
  compiling two copies of the package with incompatible types[^4]. Always import
  the lowercase path. This was one of the more disruptive incidents in the Go
  dependency ecosystem and is worth grepping your `go.mod` and transitive deps
  for.
- **`Fatal` and `Panic` are control flow, not just severity.** `log.Fatal`
  calls `os.Exit(1)` after logging (bypassing deferred functions), and
  `log.Panic` calls `panic()`. Use `RegisterExitHandler` to run cleanup before a
  fatal exit, since `os.Exit` cannot be recovered.
- **Performance on hot paths.** Field allocation and interface boxing make
  Logrus materially slower than zerolog/zap in benchmarks. For high-throughput
  logging, keep levels above the noisy ones in production, avoid fielded logging
  in tight loops, or move to a zero-allocation logger.
- **`SetNoLock` is a footgun.** Disabling the mutex is only safe when you have no
  hooks and your writer is itself concurrency-safe (e.g. an `O_APPEND` file with
  sub-4KB writes). Otherwise interleaved output and races follow.
- **No log rotation, no environment awareness.** By design. Rotation is
  delegated to external tooling like `logrotate(8)`; environment-specific
  formatter selection is left to your own `init` code.
- **Built-in test hook.** `hooks/test` (`NewNullLogger`, `NewGlobal`) lets tests
  assert on emitted entries without producing output — a genuinely useful
  facility that many alternatives lack out of the box.

## When to Use / When Not

**Use when:**
- You are maintaining an existing service already built on Logrus — it is stable
  and well understood.
- You value the mature hook and formatter ecosystem (syslog, GELF, Logstash,
  redaction) and drop-in stdlib compatibility.
- Logging volume is moderate and developer ergonomics matter more than raw
  throughput.

**Avoid when:**
- You are starting a new project — prefer the standard `log/slog`, or a
  zero-allocation logger, since Logrus takes no new features.
- Logging sits on a latency- or allocation-sensitive hot path.
- You need an actively evolving library with a roadmap.

## Alternatives

- rs/zerolog — zero-allocation JSON logger with a chained-field API; use when
  throughput and GC pressure matter.
- uber-go/zap — high-performance structured logger with typed field
  constructors; use for demanding services willing to trade some ergonomics.
- golang/go `log/slog` — the standard-library structured logger since Go 1.21;
  use for new code that wants no third-party dependency.
- apex/log — a lighter Logrus-like API with a handler model; use when you want
  Logrus's feel with a smaller surface.
- uber-go/zap `zapcore` + `slog` bridge — use when you need slog's interface
  with zap's backend.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-10 | First public commit; `WithFields` structured API[^1]. |
| — | 2017 | `Sirupsen` → `sirupsen` account rename breaks case-sensitive imports[^4]. |
| 1.0.0 | 2017 | First stable tagged release; API frozen for compatibility. |
| — | 2019 | Maintenance-mode stance formalized; README points to zerolog/zap/apex[^2]. |
| — | 2023 | Go 1.21 ships `log/slog`; Logrus focuses on interop[^3]. |
| 1.9.x | 2023–2026 | Security, bug, and performance fixes only; `interface{}` → `any` cleanups. |

## References

[^1]: Logrus README — project description, `SetReportCaller` overhead note, and usage examples. https://github.com/sirupsen/logrus/blob/master/README.md
[^2]: Logrus README, "Logrus is in maintenance mode" section — recommends Zerolog, Zap, Apex; interop with `log/slog`. https://github.com/sirupsen/logrus#readme
[^3]: Go standard library `log/slog`, introduced in Go 1.21 (2023). https://pkg.go.dev/log/slog
[^4]: Logrus issue #570 — lowercase import path guidance after the account rename. https://github.com/sirupsen/logrus/issues/570#issuecomment-313933276

## Tags

go, logging, structured-logging, observability, logrus, maintenance-mode, backend, library, mit-license, slog
