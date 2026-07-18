# uber-go/dig

> A reflection-based dependency-injection container for Go — the resolver engine underneath Uber's Fx framework.

[GitHub repo](https://github.com/uber-go/dig) ·
[Official website](https://go.uber.org/dig) ·
[License: MIT](https://github.com/uber-go/dig/blob/master/LICENSE)

## Overview

dig is a small dependency-injection library for Go, released by Uber in 2017[^1]. You register constructors (functions that return the types they build) with `Provide`, then ask for a result with `Invoke`; dig uses reflection to walk the constructor signatures, build a directed acyclic graph of type dependencies, and call constructors in the right order, caching each produced value. It is deliberately narrow: it resolves an object graph once, during process startup, and gets out of the way.

The project's own README is unusually blunt about scope. dig is "good for powering an application framework" and "bad for" being used directly by application code, for resolving dependencies after startup, or as a service locator[^1]. In practice almost nobody imports dig directly — they use uber-go/fx, the lifecycle framework built on top of it[^2]. dig is best understood as Fx's engine rather than an end-user library, which is why its star count is modest relative to its real-world footprint.

The defining tradeoff is reflection. dig resolves the graph at runtime, so there is no code generation step and no build-time coupling, but also no compile-time guarantee that the graph is complete or well-typed. A missing constructor, a cycle, or a type mismatch surfaces as a runtime error at `Invoke` time, not a compiler error. This is the fault line between dig and its main rival google/wire, which generates injector code at build time and pays zero runtime cost.

## Getting Started

```bash
go get go.uber.org/dig@v1
```

```go
package main

import (
	"log"

	"go.uber.org/dig"
)

type Config struct{ DSN string }
type DB struct{ dsn string }

func NewConfig() *Config          { return &Config{DSN: "postgres://localhost/app"} }
func NewDB(cfg *Config) *DB       { return &DB{dsn: cfg.DSN} }

func main() {
	c := dig.New()
	if err := c.Provide(NewConfig); err != nil {
		log.Fatal(err)
	}
	if err := c.Provide(NewDB); err != nil {
		log.Fatal(err)
	}

	// dig sees NewDB needs *Config, calls NewConfig first, caches both.
	err := c.Invoke(func(db *DB) {
		log.Printf("connected to %s", db.dsn)
	})
	if err != nil {
		log.Fatal(err)
	}
}
```

## Architecture / How It Works

A `Container` holds a registry keyed by type. `Provide(fn)` inspects `fn`'s signature via `reflect`: its parameters become edges into the graph, its return values become nodes the container can supply. `Invoke(fn)` does the same for the invoked function, then does a topological resolution — each required type is produced by calling its constructor (recursively resolving that constructor's own parameters first), and each produced value is memoized. Constructors therefore run at most once; every consumer of `*DB` gets the same instance. There is no per-request or transient scope at this layer — dig values are effectively singletons within a container.

Because a plain function signature can only express so much, dig adds four struct-tag mechanisms for the cases positional parameters cannot cover:

- **`dig.In`** — embed it in a struct to receive a bag of dependencies as fields instead of positional args, which keeps constructor signatures stable as dependencies grow.
- **`dig.Out`** — the symmetric form for returning many named/grouped values from one constructor.
- **Named values** (`name:"..."` tag) — disambiguate two providers of the same type (e.g. a read replica and a primary `*sql.DB`).
- **Value groups** (`group:"..."` tag) — collect many providers of a type into a slice, the idiomatic way to build plugin/handler registries.

Additional tags cover `optional:"true"` (tolerate a missing provider by receiving the zero value) and, in later versions, `dig.As` to register a concrete type under an interface, and **Scopes** (`Container.Scope`) plus **`Decorate`** to override or wrap a value within a subtree of the graph — the machinery Fx uses to implement modules. Cycles are rejected when the offending edge is added, so a well-formed container is always acyclic.

The whole design leans on `reflect`. That buys zero boilerplate and no generator in the build, at the cost of type errors that only exist at runtime and stack traces that can be hard to read.

## Production Notes

- **Errors are runtime, and historically hard to read.** A missing or mistyped dependency deep in the graph produces a chained message describing the path dig was trying to resolve. dig has invested in this (`RootCause`, provider-info hooks, better formatting), but a large Fx app's first "missing type" error is still a common onboarding wall. Failing fast at startup is the intended and correct behavior — do not defer `Invoke` errors.
- **Everything is a singleton.** dig has no transient or per-request lifetime. If you need a fresh value per operation, provide a factory (`func() *T`) and call it yourself, or use a Scope. Treating dig like a general-purpose IoC container with request scoping is a misuse the README explicitly warns against.
- **Resolution cost is startup-only.** Reflection happens while the graph is built and invoked, not on the hot path, so runtime overhead after startup is effectively nil. The cost shows up as slower, allocation-heavy startup in very large graphs — rarely a problem in practice.
- **Don't expose it as a service locator.** Passing the container around so code can pull dependencies at will defeats the point and is called out as a bad pattern[^1]. Keep `Provide`/`Invoke` at the composition root.
- **You probably want Fx, not dig.** Unless you are building a framework, uber-go/fx wraps dig with lifecycle hooks (`OnStart`/`OnStop`), logging, and module composition. Reaching for raw dig usually means reimplementing a subset of Fx.
- **Strict SemVer on v1.** The library has stayed on v1 for its whole life and promises no breaking changes to exported APIs before a v2[^1]. Upgrades within v1 are low-risk; new capabilities (scopes, decorate, callbacks) arrived as additive minor releases.

## When to Use / When Not

**Use when:**
- You are building an application framework or a composition root and want runtime graph resolution with no code-gen step.
- You want Uber's Fx and need to understand or extend the engine under it.
- Your dependency graph is assembled once at startup and then fixed.

**Avoid when:**
- You want compile-time safety and zero runtime cost — reach for google/wire.
- You need request-scoped or transient lifetimes, or dynamic re-resolution after startup — dig is not designed for it.
- You would be tempted to pass the container into business logic as a service locator.
- Your app is small enough that hand-wiring constructors in `main` is clearer than any container.

## Alternatives

- google/wire — compile-time DI via code generation; no reflection, no runtime cost, errors caught at build time. Use instead when you want static guarantees over runtime flexibility.
- uber-go/fx — the application framework built directly on dig, adding lifecycle and modules. Use instead when you want the batteries-included layer rather than the bare resolver.
- samber/do — generics-based service container with an explicit, non-reflection API. Use instead when you want a simpler, type-parameterized container for direct application use.
- goava/di / goioc/di — alternative reflection containers. Use instead only if their ergonomics fit better; they lack Fx's ecosystem.
- Plain constructor wiring in `main` — Use instead when the graph is small; no library beats a few explicit function calls for clarity.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2017-03-21 | Reflection-based container extracted at Uber[^3]. |
| 1.0.0 | 2017 | First stable release; strict SemVer v1 begins[^1]. |
| 1.x | 2018–2020 | `dig.In`/`dig.Out`, named values, value groups, optional deps. |
| 1.x (later) | 2021–2022 | Scopes and `Decorate` added — enables Fx modules. |
| — | 2025-05-13 | Last push to `master`; maintained, low-churn, still v1[^3]. |

## References

[^1]: uber-go/dig README — scope guidance, installation, SemVer v1 policy. https://github.com/uber-go/dig/blob/master/README.md
[^2]: uber-go/fx — application framework built on dig. https://github.com/uber-go/fx
[^3]: uber-go/dig repository metadata (created 2017-03-21, last push 2025-05-13). https://github.com/uber-go/dig

## Tags

go, golang, dependency-injection, di, reflection, ioc-container, uber, fx, backend, library
