# golang/go

> The Go programming language — small, statically typed, concurrent, compiled.

[GitHub repo](https://github.com/golang/go) ·
[Official website](https://go.dev) ·
[License: BSD-3-Clause](https://github.com/golang/go/blob/master/LICENSE)

## Overview

Go is a statically-typed compiled language developed at Google by Robert Griesemer, Rob Pike, and Ken Thompson, released in 2009[^1]. Its design priorities are explicit: fast compilation, simple syntax (Pike's "less is exponentially more"), built-in concurrency primitives (goroutines + channels), and a strong standard library that minimizes dependencies for common tasks.

Go's culture is anti-flexibility by design. There's one way to format code (`gofmt`), one error idiom (`if err != nil`), one module system (Go modules, after the GOPATH era), and few language features that meaningfully reduce LOC. This is widely loved and widely criticized in equal measure: critics argue Go forces verbose code; defenders argue the verbosity scales better in large teams than expressive abstractions.

**Generics** arrived in Go 1.18 (2022)[^2] — a long-debated addition that was deliberately constrained (no method-level generics, no operator overloading, no associated types). Recent Go releases focus on incremental refinement: profile-guided optimization (PGO), structured logging (`log/slog`), `range over int` / `range over func` (1.22, 1.23), and the start of "loop semantics" cleanup.

Go is dominant in: cloud infrastructure (Kubernetes, Docker, Terraform, Vault, Consul, etcd), backend services at high-scale companies (Google, Cloudflare, Uber, Twitch), CLI tools, and DevOps tooling. It is rarely chosen for: data science (no comparable ecosystem to Python), front-end work, or embedded.

## Getting Started

```bash
# macOS
brew install go

# or download from go.dev/dl
```

Initialize a module:

```bash
mkdir my-app && cd my-app
go mod init github.com/me/my-app
```

A minimal HTTP server using goroutines:

```go
package main

import (
    "encoding/json"
    "log/slog"
    "net/http"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func main() {
    http.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
        users := []User{{ID: 1, Name: "Tom"}}
        json.NewEncoder(w).Encode(users)
    })

    slog.Info("listening on :8080")
    http.ListenAndServe(":8080", nil)
}
```

```bash
go run .
```

Add a dependency:

```bash
go get github.com/google/uuid
```

## Architecture / How It Works

The Go toolchain (`gc` compiler, linker, runtime, standard library) is a single self-hosting binary. Compilation is fast by design — `gc` is intentionally not a heavy optimizing compiler. For maximum throughput, `gccgo` (the GCC frontend) produces faster code at the cost of much slower builds.

**Goroutines** are lightweight user-space threads scheduled onto OS threads by the Go runtime[^3]. A goroutine starts at ~2KB stack (grown dynamically). The scheduler (M:N model — M goroutines, N OS threads, GOMAXPROCS sets N) uses work-stealing and preemption (asynchronous, since Go 1.14).

**Channels** are typed CSP-style communication primitives. The idiom is "share memory by communicating" — channels handle most synchronization needs. Mutexes from `sync` are also available and often preferred for simple state protection.

**The garbage collector** is concurrent, tri-color, with sub-millisecond stop-the-world pauses since Go 1.5 (2015)[^4]. Recent releases have continued shrinking pause times and improving throughput. Most production Go apps never need GC tuning beyond `GOGC=100` (the default).

**Error handling.** Go uses explicit `(value, error)` return pairs. There is no `try`/`catch` (panic/recover exists for unrecoverable conditions, not control flow). The verbosity is a frequent complaint; the proposed `?` operator (try) was rejected after community discussion.

**Generics** (1.18) use bracket syntax for type parameters with constraint interfaces:

```go
func Map[T, U any](s []T, f func(T) U) []U {
    r := make([]U, len(s))
    for i, v := range s {
        r[i] = f(v)
    }
    return r
}
```

Real-world adoption is moderate — many libraries deliberately limit generic usage to keep code readable.

**Modules** (`go.mod`, `go.sum`) replaced GOPATH starting in Go 1.11 (2018) and became default in 1.16. Workspaces (`go.work`) for multi-module monorepos shipped in 1.18.

## Production Notes

**Goroutine leaks.** A goroutine that blocks on a channel or HTTP read and never receives a cancellation signal stays alive forever. Always pair long-running goroutines with `context.Context` for cancellation. Tools: `pprof` goroutine profile catches leaks at runtime.

**Context propagation.** `context.Context` is the canonical way to propagate cancellation, deadlines, and request-scoped values. Pass `ctx` as the first argument to any function that does I/O or might block. `context.WithTimeout`, `context.WithCancel`, `context.WithValue` are the primitives.

**`interface{}` → `any`.** In 1.18, `any` became a type alias for `interface{}`. New code should use `any`. Old code is fine to leave alone — they're identical at runtime.

**JSON performance.** `encoding/json` reflection-based; fine for typical loads. High-throughput services often replace it with `goccy/go-json`, `bytedance/sonic`, or the new `encoding/json/v2` (in flight). The standard library JSON v2 will be additive — `encoding/json/v1` remains.

**slog (1.21+).** `log/slog` is the standard structured logger. Replaces ad-hoc `log` and the various third-party loggers (zap, logrus). Most new services should start with `slog`; zap is still chosen when peak performance matters.

**Error wrapping.** `fmt.Errorf("context: %w", err)` wraps an error preserving the chain. `errors.Is` and `errors.As` unwrap and type-check. Pre-1.13 patterns (`pkg/errors`) are legacy.

**Profile-guided optimization (PGO).** From 1.21, `gc` can use runtime profiles (`pprof`) to optimize hot paths in subsequent builds. Typically 2–7% throughput improvement for production-tuned services.

**Range over func (1.23+).** Custom iterators via `func(yield func(T) bool)`. Most idiomatic Go uses `for ... range` over slices, maps, channels — the new range-over-func is for library authors building iterators.

**Loop variable semantics (1.22).** Pre-1.22, `for i := range xs { go f(i) }` captured a single shared `i`. From 1.22, each iteration has its own `i`. This silently changed behavior across the ecosystem; audit any pre-1.22 loop captures.

## When to Use / When Not

**Use when:**
- Building HTTP services, gRPC servers, or backend systems at scale.
- Writing CLI tools that need single-binary distribution.
- Building infrastructure software (orchestrators, schedulers, proxies).
- Your team values fast onboarding and strong conventions over expressive abstractions.

**Avoid when:**
- You're doing data science, ML, or scientific computing (Python wins).
- You need maximum single-thread performance with predictable latency (Rust, C++ win).
- Your problem benefits from sum types / pattern matching / strong type system features (Rust, OCaml, Haskell win).
- The team is small and you'd benefit from expressive features.

## Alternatives

- [rust-lang/rust](../rust-lang/rust.md) — memory safety + no GC, steeper curve, slower compile.
- **Java/Kotlin** — JVM ecosystem, GC, more mature for enterprise.
- **TypeScript on Node** — for full-stack JS teams; weaker concurrency story.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012-03 | Stable, Go 1 compatibility promise. |
| 1.5 | 2015-08 | Self-hosting compiler, concurrent GC[^4]. |
| 1.7 | 2016-08 | `context` package promoted to stdlib. |
| 1.11 | 2018-08 | Modules (experimental). |
| 1.14 | 2020-02 | Async preemption, modules stable. |
| 1.18 | 2022-03 | Generics, workspaces, fuzzing[^2]. |
| 1.21 | 2023-08 | `log/slog`, profile-guided optimization. |
| 1.22 | 2024-02 | Loop variable semantics, `range over int`. |
| 1.23 | 2024-08 | `range over func`, `unique` package. |
| 1.24 | 2025-02 | Generic type aliases stable, `weak` package. |

## References

[^1]: Rob Pike, "Less is exponentially more" — 2012. https://commandcenter.blogspot.com/2012/06/less-is-exponentially-more.html
[^2]: Go Team, "An Introduction To Generics" — 2022. https://go.dev/blog/intro-generics
[^3]: Dmitry Vyukov, "Go scheduler: implementing language with lightweight concurrency". https://github.com/golang/go/blob/master/src/runtime/HACKING.md
[^4]: Rick Hudson, "Go GC: Prioritizing low latency and simplicity" — 2015. https://go.dev/blog/go15gc
