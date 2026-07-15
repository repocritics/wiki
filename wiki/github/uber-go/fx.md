# uber-go/fx

> Reflection-based dependency injection and application lifecycle framework for Go, extracted from Uber's service stack.

[GitHub repo](https://github.com/uber-go/fx) ·
[Official website](https://uber-go.github.io/fx/) ·
[License: MIT](https://github.com/uber-go/fx/blob/master/LICENSE)

## Overview

Fx is a dependency injection (DI) container plus an application lifecycle
manager for Go. You register constructors with `fx.Provide`, ask for something
to be built with `fx.Invoke`, and Fx resolves the whole graph by reflection at
startup — wiring, ordering, and start/stop hooks included. It is the backbone
of nearly all Go services at Uber[^1], and exists to remove `init()` blocks and
global singletons from large codebases where dozens of components share
loggers, config, database handles, and tracers.

Fx is built on top of `uber-go/dig`, a lower-level reflection-based container;
Fx adds the `fx.App` runtime, lifecycle hooks, modules, and structured
logging[^2]. The defining tension is reflection: because the graph is assembled
at runtime, wiring mistakes surface as errors when the app *starts*, not when it
*compiles*. This is the exact opposite tradeoff from `google/wire`, which
generates plain Go and fails at build time. Fx trades compile-time safety and
grep-ability for flexibility, runtime value groups, and dynamic composition —
and that trade is the single most-debated thing about it.

Fx has been `v1` since 2017 and follows SemVer strictly: no breaking changes to
exported APIs before a hypothetical `v2.0.0`, which has never shipped[^3]. For a
foundational library that is a feature, not stagnation.

## Getting Started

```shell
go get go.uber.org/fx@latest
```

```go
package main

import (
	"context"
	"net/http"

	"go.uber.org/fx"
)

// A constructor: Fx will call this and cache the result as a singleton.
func NewServer(lc fx.Lifecycle) *http.Server {
	srv := &http.Server{Addr: ":8080"}
	lc.Append(fx.Hook{
		OnStart: func(ctx context.Context) error { go srv.ListenAndServe(); return nil },
		OnStop:  func(ctx context.Context) error { return srv.Shutdown(ctx) },
	})
	return srv
}

func main() {
	fx.New(
		fx.Provide(NewServer), // register the constructor (lazy)
		fx.Invoke(func(*http.Server) {}), // force it to be built + started
	).Run() // Start → wait for SIGINT/SIGTERM → Stop
}
```

`fx.New` builds the graph, `Run` starts all lifecycle hooks in dependency
order, blocks on a signal, then runs `OnStop` hooks in reverse (LIFO).

## Architecture / How It Works

The container is lazy: a constructor registered via `fx.Provide` is only called
if something reachable from an `fx.Invoke` actually needs its output. Each
provided type is a singleton within the app — Fx caches the first result and
reuses it.

Wiring beyond simple positional arguments uses **parameter and result objects**:

- `fx.In` / `fx.Out` — embed these in a struct to receive or supply multiple
  dependencies as fields, with struct tags controlling behavior. As of v1.14
  these are type aliases rather than distinct structs[^4].
- `name:"..."` — distinguish two values of the same type.
- `optional:"true"` — tolerate a missing dependency (zero value if absent).
- `group:"..."` — **value groups**: many providers contribute to a slice
  consumed as `[]T`. Ordering within a group is not deterministic.

Higher-level composition tools sit on top of the raw container:

- **`fx.Module`** — a named, scoped subgraph. Provides inside a module are
  private to it unless explicitly exported, which is the main tool for keeping
  large graphs from becoming one flat global namespace.
- **`fx.Decorate`** — wrap or replace a dependency for a scope (e.g. inject a
  test double or add middleware around a real value).
- **`fx.Supply`** — inject an already-constructed value instead of a constructor.
- **`fx.Annotate`** — attach group/name tags or `fx.As` interface bindings to an
  otherwise plain constructor without writing an `fx.In`/`fx.Out` struct.
- **`fxevent`** — a structured event stream (v1.14) that powers logging; swap
  the default JSON logger via `fx.WithLogger`, or route it to Zap[^5].

Fx can emit the resolved graph as Graphviz DOT (`fx.DotGraph`) and annotates
resolution failures with the relevant subgraph, which is the practical answer to
"what depends on what" in a large app.

## Production Notes

**Reflection cost is a startup cost, not a request cost.** All the reflective
graph resolution happens once during `fx.New` / `Start`. There is no per-request
DI overhead — resolved singletons are ordinary Go values on the hot path. The
startup penalty is real but bounded and usually irrelevant next to opening
database and network connections.

**Errors are runtime, and historically cryptic.** A missing provider, a cycle,
or a type mismatch fails when the app starts. Message quality has improved
substantially across versions (subgraph context, DOT export, timeout details),
but deep graphs still produce error text that takes practice to read. This is
the recurring complaint from teams evaluating Fx against `google/wire`.

**Grep-ability and IDE navigation suffer.** Because the connection between "who
provides `*sql.DB`" and "who consumes it" is made by type at runtime, jump-to-
definition and plain text search do not reveal the wiring. New engineers on a
large Fx codebase routinely report that tracing data flow is harder than in
hand-wired code. Disciplined use of `fx.Module` mitigates but does not remove
this.

**Lifecycle hooks have timeouts and ordering rules.** `OnStart`/`OnStop` run
under a context with a deadline; a hook that blocks past it fails the start.
`OnStart` must not block the goroutine (spawn servers in a goroutine, as in the
example above) — earlier versions could deadlock if a hook exited the current
goroutine, fixed in v1.18[^6]. Stop order is strict LIFO.

**Adopt at the top, not in libraries.** Fx is an application framework. A
reusable library should never force its consumers to use Fx; expose plain
constructors and let the application decide. Fx-flavored `Module`s are the
sanctioned way to publish shareable components without leaking the framework
into every import.

## When to Use / When Not

**Use when:**
- You run many long-lived services that share a common set of components
  (logger, config, tracer, DB pools) and want them wired consistently.
- Multiple teams publish and consume shared modules across services.
- You want managed start/stop lifecycles with ordered, timeout-bounded hooks.
- You are already inside the Uber-go ecosystem (Zap, dig, multierr).

**Avoid when:**
- The app is small enough that a few explicit constructors in `main.go` are
  clearer — manual wiring beats magic below a certain size.
- You want wiring errors at compile time; reach for `google/wire` instead.
- You're writing a library — never impose an application framework on callers.
- Your team values grep-able, statically traceable data flow over composition
  flexibility.

## Alternatives

- google/wire — compile-time DI via code generation; errors at build time, zero
  runtime reflection. Use when you want static guarantees and no startup magic.
- uber-go/dig — the lower-level reflection container Fx is built on. Use when you
  want DI resolution but not the app lifecycle, modules, or logging layer.
- samber/do — generics-based DI (Go 1.18+), lighter and more type-explicit. Use
  for smaller apps that want structure without full reflection.
- google/guice-style manual wiring in `main.go` — use when the graph is small
  enough that explicit constructor calls are the most readable option.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2017-07-31 | First stable release; strict SemVer begins[^3]. |
| v1.13.0 | 2020-06-16 | Late pre-modules era; value groups, annotations maturing. |
| v1.14.0 | 2021-08-12 | `fxevent` structured events, `fx.WithLogger`; `fx.In`/`fx.Out` become type aliases[^4]. |
| v1.18.0 | 2022-08-08 | Soft value groups, `OnStart`/`OnStop` annotations, lifecycle deadlock fixes[^6]. |
| v1.20.0 | 2023-06-12 | `fxevent.Run` event for constructor/decorator runs[^7]. |
| v1.23.0 | 2024-10-11 | Continued maintenance release. |
| v1.24.0 | 2025-05-13 | Latest tagged release as of this writing[^8]. |

## References

[^1]: Fx README, "Battle tested" — Fx is the backbone of nearly all Go services at Uber. https://github.com/uber-go/fx
[^2]: uber-go/dig, the reflection-based container Fx builds upon. https://github.com/uber-go/dig
[^3]: Fx README, "Stability" — v1, strict SemVer, no breaking changes before v2.0.0. https://github.com/uber-go/fx#stability
[^4]: Fx v1.14.0 release notes — `fxevent` package, `fx.WithLogger`, `fx.In`/`fx.Out` as type aliases. https://github.com/uber-go/fx/releases/tag/v1.14.0
[^5]: Fx documentation — logging and Zap integration. https://uber-go.github.io/fx/
[^6]: Fx v1.18.0 release notes — soft value groups, OnStart/OnStop annotations, lifecycle deadlock fix. https://github.com/uber-go/fx/releases/tag/v1.18.0
[^7]: Fx v1.20.0 release notes — `fxevent.Run` event. https://github.com/uber-go/fx/releases/tag/v1.20.0
[^8]: Fx releases (v1.24.0, 2025-05-13). https://github.com/uber-go/fx/releases

## Tags

go, golang, dependency-injection, application-framework, di-container, reflection, lifecycle-management, microservices, uber-go, backend
