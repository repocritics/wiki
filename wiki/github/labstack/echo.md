# labstack/echo

> A minimalist Go web framework layered on `net/http`: radix-tree routing, binding, middleware, and centralized error handling — without leaving the standard library behind.

[GitHub repo](https://github.com/labstack/echo) ·
[Official website](https://echo.labstack.com) ·
[License: MIT](https://github.com/labstack/echo/blob/master/LICENSE)

## Overview

Echo is one of the two long-standing "batteries-included but thin" Go web
frameworks (the other being Gin). First published by Vishal Rana in 2015[^1], it
sits directly on Go's standard `net/http` rather than replacing it — a handler is
still fed by a standard server, and Echo interoperates both ways through
`echo.WrapHandler` and `echo.WrapMiddleware`. What it adds is the set of pieces
`net/http` deliberately leaves to the user: a fast radix-tree router with route
prioritization, request binding for JSON/XML/form/query, a pluggable validator
hook, a broad middleware ecosystem, and a single funnel for HTTP error handling.

With ~32.5k stars and ~2.3k forks it is a mainstream, mature choice rather than a
frontier one — the API surface has been stable across long stretches, and the
project has settled into a maintenance-and-refinement cadence under a small team
(Roland Lammel, Martti T., Pablo Andres Fuente) after original author Vishal
Rana. The defining tension is philosophical: Echo stays close to `net/http` and
the standard `context.Context`/`slog` world, which makes it composable and
future-proof, but that same restraint means it does less magic than Gin's
ecosystem and is measurably slower than `fasthttp`-based frameworks like Fiber
that abandon `net/http` for throughput.

The current release line is `v5`, cut 2026-01-18[^2]; `v4` remains supported with
security and bug fixes until 2026-12-31[^2]. Because Go modules encode the major
version in the import path (`github.com/labstack/echo/v4` vs `.../v5`), upgrades
are explicit and opt-in — a property that matters a lot in the Production Notes
below.

## Getting Started

```sh
go get github.com/labstack/echo/v5
```

```go
package main

import (
	"log/slog"
	"net/http"

	"github.com/labstack/echo/v5"
	"github.com/labstack/echo/v5/middleware"
)

func main() {
	e := echo.New()
	e.Use(middleware.RequestLogger()) // structured request logging via slog
	e.Use(middleware.Recover())       // convert panics into handled errors

	e.GET("/", func(c *echo.Context) error {
		return c.String(http.StatusOK, "Hello, World!")
	})

	if err := e.Start(":8080"); err != nil {
		slog.Error("server exited", "error", err)
	}
}
```

Echo supports the last four Go major releases and may work with older ones[^1].

## Architecture / How It Works

The core is an `Echo` instance holding a **radix (prefix) tree** router. On each
request the router walks the tree to resolve the handler and path parameters,
choosing between static, parameterized (`:id`), and wildcard (`*`) segments by a
fixed priority (static > param > any). The router is allocation-conscious: path
parameters are stored in a preallocated slice on the context rather than a map, a
design Echo shares with Gin and which accounts for much of its routing speed.

Requests are wrapped in a **`Context`** that carries the request/response, path
params, bound data, and per-request storage (`Get`/`Set`). A pivotal v5 change:
`Context` moved from an *interface* (`echo.Context`) to a *concrete struct*
handlers receive by pointer (`*echo.Context`)[^2]. This removes an interface
indirection and simplifies extension, but it is a breaking change that touches
every handler and middleware signature — the single largest reason a v4→v5 bump
is not mechanical.

Contexts are **pooled** via `sync.Pool` and reused across requests. This is the
framework's most important and most-violated invariant: a `Context` (and anything
derived from its request/response) must not be retained past the return of the
handler. Spawning a goroutine that closes over `c` and outlives the handler is a
use-after-free-shaped bug that surfaces as data races or corrupted responses
under load.

**Middleware** are `func(next HandlerFunc) HandlerFunc` closures, registered at
three scopes — root (`e.Use`), group (`e.Group(...).Use`), and per-route — and
executed in registration order around the handler. **Groups** share a path prefix
and a middleware stack, which is how most apps carve out `/api/v1` or
authenticated sections. **Binding** (`c.Bind`) inspects `Content-Type` to decode
JSON/XML/form/query into a struct; validation is intentionally *not* built in —
you register a `Validator` (commonly a wrapper over `go-playground/validator`).
**Error handling** is centralized: handlers return `error`, and a single
`HTTPErrorHandler` maps those (including `*echo.HTTPError`) to responses, so
error-to-status logic lives in one place instead of every handler.

Batteries that ship in-tree include automatic TLS via Let's Encrypt (autocert),
HTTP/2, static file serving, and template rendering against any engine through a
`Renderer` interface. Official middleware for JWT, OpenTelemetry, Prometheus, and
session/casbin integrations live in separate `labstack/echo-*` repos[^1].

## Production Notes

- **Never retain the `Context`.** Because contexts are `sync.Pool`-recycled, any
  work that outlives the handler (background goroutines, deferred logging that
  reads the request later) must copy the values it needs first. This is the most
  common Echo production bug and it only manifests under concurrency.
- **Validation is your job.** `c.Bind` populates the struct but does not validate
  it; forgetting to wire a `Validator` means malformed input passes silently.
  Bind also does not, by itself, defend against JSON parameter-pollution or
  oversized bodies — pair it with `middleware.BodyLimit` and explicit checks.
- **Static/file serving has been a security-sensitive surface.** Path-handling in
  static file serving is exactly the class of code that has drawn CVEs across Go
  web frameworks; keep Echo patched to the latest `v4`/`v5` point release rather
  than pinning an old minor, and avoid serving user-controlled paths without
  sanitization.
- **Major versions are separate import paths.** `v3`, `v4`, and `v5` are distinct
  modules; a large codebase can accidentally depend on two at once (directly and
  transitively), doubling the router in memory and confusing middleware. Audit
  `go mod graph` for mixed `echo` majors.
- **The v4→v5 migration is real work.** The interface→struct `Context` change,
  the move to `slog`-based logging, and other API adjustments mean v5 is not a
  drop-in — budget for touching every handler signature and consult
  `API_CHANGES_V5.md`[^2]. With v4 supported only through 2026-12-31, this is a
  scheduled, not optional, migration.
- **Performance is good, not best-in-class.** Echo's router is fast and low-alloc,
  competitive with Gin. If your bottleneck is genuinely HTTP framework overhead
  (rare for real apps doing I/O), a `fasthttp` framework will beat it — at the
  cost of `net/http` ecosystem compatibility.
- **Logging changed.** v5 leans on the standard library `log/slog`; teams coming
  from Echo's older bespoke logger, or from Zap/Zerolog wrappers, should confirm
  their logging middleware still matches the new interfaces.

## When to Use / When Not

**Use when:**
- You want structured routing, binding, groups, and centralized errors without
  leaving `net/http` and standard `context.Context`/`slog` behind.
- You're building conventional REST/JSON services or microservices and value a
  stable, well-trodden API over cutting-edge throughput.
- You need built-in autocert TLS, HTTP/2, and a maintained middleware catalog out
  of the box.

**Avoid when:**
- Raw requests-per-second is the dominant constraint and you'll accept dropping
  `net/http` compatibility — a `fasthttp` framework (Fiber) fits better.
- You want the largest community and plugin surface — Gin still has more
  third-party middleware and StackOverflow coverage.
- Your routing needs are modest and you'd rather have zero framework dependency —
  Go 1.22+ `net/http.ServeMux` now does method- and wildcard-based routing.

## Alternatives

- gin-gonic/gin — the most popular Go framework with a similar radix router; use it when community size and middleware availability outweigh Echo's tidier error model.
- gofiber/fiber — `fasthttp`-based, Express-like API; use it when maximum throughput matters more than `net/http` interoperability.
- go-chi/chi — idiomatic `net/http` router with `http.Handler` middleware; use it when you want composability and stdlib compatibility without a framework `Context`.
- gorilla/mux — mature, stdlib-compatible router; use it when you need only routing (no binding/rendering) and value long-term stability.
- golang/go (`net/http.ServeMux`) — since Go 1.22 supports method and wildcard patterns; use it when stdlib routing is enough and you want zero dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2015 | Initial release; radix-tree router on top of `net/http`[^1]. |
| 3.x | 2017 | Import path `github.com/labstack/echo`; middleware ecosystem matures. |
| 4.0 | 2019 | Go modules; import path moves to `.../v4`; standard `context` alignment. |
| 5.0 | 2026-01-18 | `Context` becomes a concrete `*echo.Context` struct; `slog`-based logging; see `API_CHANGES_V5.md`[^2]. |

## References

[^1]: Echo README and project overview, labstack/echo. https://github.com/labstack/echo
[^2]: Echo supported-versions notice and v5 changes (`API_CHANGES_V5.md`, `ROADMAP.md`); v5 released 2026-01-18, v4 supported through 2026-12-31. https://github.com/labstack/echo/blob/master/API_CHANGES_V5.md

## Tags

go, web-framework, http, rest-api, net-http, router, middleware, microservices, backend, minimalist
