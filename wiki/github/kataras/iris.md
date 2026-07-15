# kataras/iris

> A batteries-included Go web framework built around a large `Context` interface, an expressive macro router, and reflection-based MVC/DI — capable but single-maintainer, with a contested history in the Go community.

[GitHub repo](https://github.com/kataras/iris) ·
[Official website](https://www.iris-go.com) ·
[License: BSD-3-Clause](https://github.com/kataras/iris/blob/main/LICENSE)

## Overview

Iris is a full-featured HTTP web framework for Go, first published in 2016 by Gerasimos "kataras" Maropoulos[^1]. Unlike the minimalist Go frameworks that expose thin wrappers over `net/http`, Iris ships an integrated stack: a macro-based router, MVC, dependency injection, sessions, view engines, i18n, websockets (via kataras/neffos), gRPC integration, and a large middleware collection — all under one module. Its stated positioning is "the fastest Go web framework" with "lifetime active maintenance."

The framework is organized around a single rich `iris.Context` interface. Handlers have the signature `func(ctx iris.Context)`, and nearly all request/response behavior — binding, content negotiation, sessions, streaming, view rendering — is reached through methods on that one object. This makes simple apps concise but produces a very wide abstraction surface that diverges from the idiomatic `http.Handler` / `http.HandlerFunc` shape most of the Go ecosystem composes around.

Iris is best understood through its defining tension: it is technically broad and actively developed, but it is effectively a one-person project with a reputation problem. Several git-history rewrites in its early years broke `go get` for downstream users, and its self-published "fastest framework" benchmarks have long been disputed[^2]. Many Go developers accordingly steer newcomers toward gin, echo, chi, or the standard library. Whether that reputation is still deserved is a judgment call; the technical capability is real, the bus-factor and governance concerns are also real.

## Getting Started

```bash
go get github.com/kataras/iris/v12@latest
```

Note the `/v12` module-path suffix — it is mandatory. Importing `github.com/kataras/iris` (without the version) targets pre-modules code and will not resolve the current release.

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.New()

    // Macro router: {name:string} is a typed, validated path parameter.
    app.Get("/hello/{name:string}", func(ctx iris.Context) {
        ctx.JSON(iris.Map{"message": "Hello " + ctx.Params().Get("name")})
    })

    // Route grouping ("Party") + middleware.
    users := app.Party("/users")
    users.Use(func(ctx iris.Context) { ctx.Next() })
    users.Get("/{id:uint64}", func(ctx iris.Context) {
        id, _ := ctx.Params().GetUint64("id")
        ctx.JSON(iris.Map{"id": id})
    })

    app.Listen(":8080")
}
```

## Architecture / How It Works

**Router and macros.** Iris uses a trie-based router with a "macro interpreter": path segments like `{id:uuid}`, `{n:int min(1)}`, or `{file:path}` compile into typed validators at build time. The type name maps to a parameter parser, and constraints (`min`, `max`, `regexp`, custom funcs) become runtime checks that reject non-matching requests before the handler runs. This is more expressive than the plain `:param` routing in most Go routers and is one of Iris's genuine differentiators. Since Go 1.22 added method+pattern matching to `net/http.ServeMux`, the baseline case is less special, but Iris's typed constraints still go further.

**Context as the core.** `iris.Context` is a large interface implemented by a pooled concrete type (`sync.Pool`-backed to reduce allocations per request). Middleware advances the chain with `ctx.Next()`; flow control uses `ctx.StopExecution()` / `ctx.StopWithError()`. Because the framework does not hand you a raw `http.ResponseWriter`/`*http.Request` by default, interop with standard-library middleware goes through adapters (`iris.FromStd`).

**MVC and dependency injection.** The `mvc` package is built on the internal `hero` DI container. Controllers are structs whose exported methods map to routes by naming convention (`GetBy`, `PutBy`, …), and dependencies — services, request-bound structs, path params — are supplied by reflection at registration time. Handler input/output arguments can likewise be injected and their return values auto-serialized. This is powerful for larger applications but is reflection-heavy: less compile-time safety, harder-to-trace wiring, and a runtime cost relative to hand-written handlers.

**Everything else in-tree.** Sessions, view engines (HTML, Django/pongo2, Handlebars, Pug, Ace, Blocks), i18n, sitemap, request/response compression (gzip, deflate, brotli, snappy, s2), content negotiation, websockets, and gRPC bridging are all first-party. Convenience is high; the dependency footprint and coupling to Iris-specific abstractions are correspondingly high.

## Production Notes

**Single-maintainer bus factor.** Iris is overwhelmingly the work of one author. Releases are frequent and the changelog (`HISTORY.md`) is detailed, but institutional continuity, security response, and review depth are what you would expect from a solo project rather than a foundation-backed one. Weigh this for anything long-lived.

**Import-path discipline.** Always import `github.com/kataras/iris/v12`. Historic git-history rewrites and version churn mean stale tutorials frequently show broken import paths; copy the module path from the current README, not from old blog posts.

**Benchmark claims are self-published.** The "fastest" marketing derives from the maintainer's own `server-benchmarks` repo. In real services, router overhead is dwarfed by JSON (de)serialization, database round-trips, and application logic; framework choice rarely moves production p99 latency meaningfully. Treat the ranking as directional at best and benchmark your own workload.

**Reflection cost surfaces at scale.** The MVC/DI path trades throughput and startup clarity for ergonomics. Hot paths that matter should be measured; you can always drop to plain `func(iris.Context)` handlers (or lower) for the routes that count.

**Ecosystem interop.** Because Iris does not expose the standard `http.Handler` contract by default, the large body of `net/http`-shaped middleware (from chi, alice, gorilla, etc.) does not drop in directly. Budget for adapter glue or Iris-native equivalents.

**Upgrades.** The v12 line has been stable in its import path since Go modules adoption, but Iris historically shipped sweeping major-version rewrites (v6, v7, v8… each with breaking API changes). Pin versions, read `HISTORY.md` before bumping, and expect meaningful churn across majors.

## When to Use / When Not

**Use when:**
- You want one framework that already bundles MVC, DI, sessions, i18n, views, websockets, and compression without assembling them.
- You value the typed macro router's path constraints and content-negotiation helpers.
- A solo/small team prioritizes breadth of built-in features and fast scaffolding over ecosystem-standard composition.

**Avoid when:**
- You want to compose the `net/http` middleware ecosystem or keep close to the standard library.
- Long-term governance and multi-maintainer continuity are hard requirements.
- You prefer explicit, reflection-free wiring for traceability and compile-time safety.
- You are choosing on the basis of the "fastest" claim alone — the real-world difference versus gin/echo/chi is usually negligible.

## Alternatives

- gin-gonic/gin — the most popular Go web framework; smaller surface, huge middleware ecosystem. Use instead when you want community mass and `net/http` interop.
- labstack/echo — comparable feature scope to Iris with a more conventional API. Use instead when you want batteries-included ergonomics with wider adoption.
- go-chi/chi — idiomatic `net/http.Handler` composition, no framework lock-in. Use instead when you want stdlib-compatible routing and middleware.
- gofiber/fiber — fasthttp-based, high throughput, non-`net/http`. Use instead when raw benchmark numbers are a genuine priority (accepting the fasthttp ecosystem trade-off).
- Standard library `net/http` — since Go 1.22, method + wildcard routing is built in. Use instead when your needs are modest and you want zero dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-01 | First public release; rapid breaking rewrites through early majors[^1]. |
| v6–v8 | 2017 | Version/import-path churn; git-history rewrites broke downstream `go get`; benchmark and community disputes[^2]. |
| v11 | 2018 | Consolidation of the pre-modules API. |
| v12.0 | 2019 | Go modules adoption; versioned `/v12` import path. |
| v12.2.0 | 2023-03-11 | Feature release; reaffirmed "lifetime active maintenance"[^3]. |
| v12.2.x | 2024–2026 | Ongoing patch/feature releases on the v12 line. |

## References

[^1]: kataras/iris repository and README. https://github.com/kataras/iris
[^2]: Community discussion of Iris's history rewrites and benchmark disputes is extensive across Go forums, Hacker News, and Reddit; the self-published benchmarks live at https://github.com/kataras/server-benchmarks
[^3]: Iris `HISTORY.md`, v12.2.0 entry (11 March 2023). https://github.com/kataras/iris/blob/main/HISTORY.md

## Tags

go, golang, web-framework, http, mvc, dependency-injection, router, websocket, sessions, backend, api
