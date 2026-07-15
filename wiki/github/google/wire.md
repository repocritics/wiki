# google/wire

> Compile-time dependency injection for Go — a code generator that writes the boilerplate initialization you would otherwise write by hand.

[GitHub repo](https://github.com/google/wire) ·
[License: Apache-2.0](https://github.com/google/wire/blob/main/LICENSE)

## Overview

Wire is a code-generation tool from Google that automates wiring together an application's object graph[^1]. You declare dependencies as ordinary function parameters; Wire reads them at generate time and emits a plain Go function that calls your constructors in the correct order. It produces no runtime container, uses no reflection, and holds no runtime state — the generated code is exactly what a careful engineer would write by hand, so it reads and debugs like normal Go.

The defining tension is *generate-time vs. run-time* dependency injection. Reflection-based containers (uber-go/dig, uber-go/fx) resolve the graph while the program runs, which is flexible but pushes wiring errors to startup and makes the resolution path opaque. Wire moves that resolution to `go generate`: a missing or ambiguous provider is a build failure with a file-and-line message, and the output is inspectable Go source you commit to the repo. The cost is a mandatory codegen step and a more rigid model — the graph is fixed at compile time, so anything genuinely dynamic (choosing an implementation from a runtime config value) must be expressed as a provider you write yourself.

Wire has been feature-complete and in beta since v0.3.0, by the maintainers' own description[^2], and in 2025 the repository was archived and explicitly marked no longer maintained[^3]. It is stable and widely used in existing Go codebases (it originated alongside the go-cloud project), but new adopters should read it as a finished, frozen tool rather than an evolving one.

## Getting Started

```shell
go install github.com/google/wire/cmd/wire@latest
```

Write an *injector*: a stub function whose body is a single `wire.Build` call listing the providers. The `wireinject` build tag keeps this file out of normal builds.

```go
//go:build wireinject
// +build wireinject

package main

import "github.com/google/wire"

// InitializeApp is the injector. Wire replaces its body.
func InitializeApp() (*App, error) {
    wire.Build(NewApp, NewGreeter, NewMessage)
    return nil, nil // arguments are ignored; only the types matter
}
```

Running `wire` in that package generates `wire_gen.go`:

```go
//go:build !wireinject
// +build !wireinject

func InitializeApp() (*App, error) {
    message := NewMessage()
    greeter := NewGreeter(message)
    app := NewApp(greeter)
    return app, nil
}
```

You commit `wire_gen.go` and call `InitializeApp()` from `main`. Re-run `wire` (or `go generate`) whenever constructors change.

## Architecture / How It Works

Wire has exactly two concepts:

- **Providers** — ordinary functions that return a value (optionally with an `error` and/or a cleanup function). A provider's parameters declare what it depends on; its return type declares what it produces. Wire matches providers to consumers *by type*, so the type graph is the dependency graph.
- **Injectors** — functions you stub out with a `wire.Build(...)` call. Wire performs a topological sort of the listed providers and emits a real function body that instantiates each dependency once and threads it through.

Supporting primitives layer onto those two:

- `wire.NewSet(...)` groups related providers into a reusable *provider set*, so a package can export its wiring as one symbol.
- `wire.Bind(new(Iface), new(*Impl))` tells Wire to satisfy an interface parameter with a concrete type — the mechanism for programming to interfaces.
- `wire.Struct(new(T), "*")` builds a struct by field injection instead of a constructor.
- `wire.Value` / `wire.InterfaceValue` inject constant or externally-supplied values.

Because matching is purely by type, **two providers of the same type in one injector is a compile-time error** — Wire cannot guess which you meant. The idiomatic fix is a named type (`type Primary *sql.DB`) so the two databases are distinct in the graph. This type-identity model is the whole design: it is what makes the tool reflection-free and the output trivial, and also what makes some real-world graphs feel verbose.

Cleanup is first-class. A provider may return `(T, func(), error)`; Wire collects the `func()` cleanups and, in the generated injector, wires them so that if a later provider fails, everything already constructed is torn down in reverse order. This is one of the few places the generated code is non-obvious, and a genuine convenience over hand-wiring.

## Production Notes

- **The archive is the headline operational fact.** Wire is marked no longer maintained[^3]; there will be no upstream fixes for new Go versions or edge cases. It is a mature, small, well-understood tool, but you are adopting a frozen dependency. Plan to fork if you need changes.
- **`wire_gen.go` is source you own.** It is generated but committed. If a teammate edits a constructor and forgets to re-run `wire`, the build still succeeds against *stale* wiring until types stop matching. Enforce regeneration in CI (`wire ./... && git diff --exit-code`) to catch drift.
- **Generate-time errors are good; they are also all you get.** Wire will not help with values only known at runtime. Feature flags, plugin selection, or "use Redis in prod, in-memory in tests" are expressed by writing a provider that makes the choice, not by Wire deciding — the graph shape is fixed once generated.
- **Same-type collisions scale badly.** Large graphs with several `*sql.DB`, `*http.Client`, or `string` config values force a proliferation of named wrapper types. This is by design but is the most common source of friction in big injectors.
- **Interfaces need `wire.Bind` everywhere.** Wire does not automatically satisfy an interface parameter from a concrete provider; forgetting the bind yields a "no provider found" error. Centralizing binds in a provider set reduces the repetition.
- **Build-tag hygiene.** The injector file and the generated file rely on the `wireinject` / `!wireinject` tag pair. Editors and linters occasionally mishandle the excluded file; keep the tags exact (both the `//go:build` and legacy `// +build` lines) to avoid duplicate-symbol errors.

## When to Use / When Not

**Use when:**
- You want dependency injection but reject runtime reflection and hidden containers — you want to read the exact initialization code.
- You value startup errors surfacing at build time instead of at process boot.
- Your object graph is largely static and known at compile time.
- You are already in an ecosystem (go-cloud and similar Google-adjacent Go projects) where Wire is the established pattern.

**Avoid when:**
- You need runtime flexibility — dynamic provider selection, hot-swapping, or plugin graphs decided from config at boot.
- You want lifecycle management (start/stop hooks, ordered shutdown, health) baked in — that is uber-go/fx's territory, not Wire's.
- You would rather not commit generated code or run a codegen step in your build pipeline.
- You need an actively maintained tool with support for future Go language changes.

## Alternatives

- uber-go/fx — runtime DI plus an application lifecycle framework (start/stop hooks, modules); use it when you want managed lifecycle and accept reflection at boot.
- uber-go/dig — the reflection-based container underneath fx; use it when you want a runtime graph without fx's app scaffolding.
- samber/do — generics-based runtime DI container; use it for a lightweight modern runtime option with minimal boilerplate.
- google/guice — the Java DI framework Wire's philosophy contrasts with; relevant only as conceptual lineage, not a Go option.
- Hand-written initialization — for small services the honest alternative; Wire's own docs note its output is just the code you'd write yourself, so tiny graphs may not justify the tool.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2018-12 | First tagged release; originated alongside go-cloud[^1]. |
| v0.3.0 | 2019 | Declared beta and feature-complete; the maintainers stopped accepting new features[^2]. |
| v0.4.0–v0.5.0 | 2019–2021 | Bug fixes and maintenance; API kept intentionally small. |
| v0.6.0 | 2024 | Added support for Go generics in providers. |
| archived | 2025-08 | Repository archived, marked no longer maintained[^3]. |

## References

[^1]: Robert van Gent, "Wire: Automated Initialization in Go" — The Go Blog. https://go.dev/blog/wire
[^2]: Wire README, "Project status" — beta and feature complete as of v0.3.0. https://github.com/google/wire#project-status
[^3]: Wire README maintenance notice, "This project is no longer maintained"; repository `archived: true` via GitHub API. https://github.com/google/wire

## Tags

go, golang, dependency-injection, code-generation, compile-time, codegen, initialization, wiring, unmaintained, google
