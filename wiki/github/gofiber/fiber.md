# gofiber/fiber

> An Express-style web framework for Go, built on fasthttp instead of net/http — trading standard-library compatibility for raw throughput.

[GitHub repo](https://github.com/gofiber/fiber) ·
[Official website](https://gofiber.io) ·
[License: MIT](https://github.com/gofiber/fiber/blob/main/LICENSE)

## Overview

Fiber is a Go web framework whose API is deliberately modeled on Express.js: `app.Get`, `app.Use`, chained middleware, and a single mutable request/response context (`fiber.Ctx`)[^1]. It was created in 2020 and has grown into one of the most-starred Go web frameworks. Its target audience is teams that want Node-flavored ergonomics in Go and prioritize request throughput for JSON APIs.

The defining decision — and the source of nearly every tradeoff below — is that Fiber is built on `valyala/fasthttp` rather than the standard library's `net/http`[^2]. fasthttp achieves its speed by pooling and reusing request/response objects and by returning byte slices that point into reusable buffers. This is what lets Fiber advertise low allocation counts, but it also means Fiber lives in a parallel universe from the rest of the Go HTTP ecosystem: standard `http.Handler` middleware, `context.Context` plumbing, and most observability tooling do not plug in directly.

The practical consequence is a framework that is fast and pleasant for its happy path, but one where "just use the library from the Go ecosystem" frequently does not apply. You are opting into fasthttp's constraints along with Fiber's conveniences.

## Getting Started

```bash
go get github.com/gofiber/fiber/v2
```

```go
package main

import "github.com/gofiber/fiber/v2"

func main() {
	app := fiber.New()

	app.Get("/users/:id", func(c *fiber.Ctx) error {
		id := c.Params("id")
		return c.JSON(fiber.Map{"id": id})
	})

	// c.Params returns a slice into a pooled buffer — copy it if
	// you retain it past the handler's return.
	app.Listen(":3000")
}
```

## Architecture / How It Works

Fiber is a thin, opinionated layer over fasthttp. The request lifecycle is:

1. fasthttp accepts the connection and populates a `RequestCtx` from a `sync.Pool`.
2. Fiber wraps it in a `fiber.Ctx` (also pooled), runs the matched middleware chain, and releases both back to their pools when the handler returns.
3. The router matches on a radix-tree structure designed to avoid heap allocation on the hot path.

Key internals worth understanding:

- **Context reuse is the central footgun.** `fiber.Ctx` and the values it returns (`c.Params`, `c.Query`, `c.Body`, header values) are only valid for the duration of the handler. After the handler returns, the underlying buffer is reused by another request. Any value you keep — in a goroutine, a channel, a struct field — must be copied first (`utils.CopyString`, or `c.Params("id", "")` semantics documented per method)[^3]. This has caused a long tail of data-corruption bugs in Fiber applications.
- **net/http bridge.** The `adaptor` package converts between `http.Handler`/`http.HandlerFunc` and Fiber handlers, but every conversion copies the request/response, erasing the fasthttp performance advantage for that route[^4].
- **Middleware.** Fiber ships a large set of first-party middleware (logger, recover, CORS, compression, rate-limiter, session, cache) rather than relying on the ecosystem, precisely because ecosystem middleware is net/http-shaped.
- **Prefork.** An optional mode forks multiple processes bound to the same port via `SO_REUSEPORT`, used to spread load across cores. It changes process semantics (multiple PIDs, per-process in-memory state) and interacts badly with in-process caches and graceful-shutdown assumptions.

The router, context, and middleware are all co-designed around fasthttp's zero-copy model. That coupling is the whole point — and the whole constraint.

## Production Notes

**No HTTP/2 (or HTTP/3).** fasthttp does not implement HTTP/2, so Fiber does not serve it[^2]. If you need HTTP/2 (gRPC, server push, some CDN/edge requirements) you terminate it at a reverse proxy in front of Fiber, or you do not use Fiber. This is the single most common surprise for teams evaluating it.

**Buffer-reuse bugs are the recurring incident class.** The most frequent production failure is retaining a `c.Params`/`c.Query`/header string past the request and seeing it mutate under load. It is invisible in single-request testing and only appears under concurrency. Treat every string off `fiber.Ctx` as borrowed, not owned.

**Ecosystem integration costs extra.** OpenTelemetry, structured tracing, and many auth/database middleware assume `net/http`. Fiber has first-party or community equivalents, but you are choosing from a smaller pool and sometimes wrapping through `adaptor` at a cost.

**Error handling is centralized.** Handlers return `error`; a single configurable `ErrorHandler` on the app decides the response. This is clean but differs from the `net/http` convention and means panics-vs-errors discipline matters (`recover` middleware is not on by default).

**v2 → v3 is a breaking migration.** Fiber 3 restructured parts of the context and middleware APIs and changed several defaults; it is not a drop-in bump from v2[^5]. The import path is versioned (`/v2`, `/v3`), so both can coexist during migration, but plan for handler-signature and middleware-config changes.

## When to Use / When Not

**Use when:**
- You want Express-like ergonomics in Go and your team already thinks in that model.
- Your workload is high-volume JSON/REST over HTTP/1.1 and throughput matters.
- You are comfortable staying inside Fiber's first-party middleware set.

**Avoid when:**
- You need HTTP/2 or gRPC served directly by the app.
- You want to reuse the broad `net/http` middleware and tooling ecosystem.
- You value standard-library idioms and `context.Context` plumbing over raw speed.
- Your team is prone to holding request-scoped values past the handler and cannot enforce the copy discipline.

## Alternatives

- gin-gonic/gin — the most popular Go web framework; built on net/http, huge middleware ecosystem. Use instead when standard-library compatibility matters more than peak throughput.
- labstack/echo — similar ergonomics to Fiber but on net/http. Use when you want Fiber-like routing without the fasthttp constraints.
- go-chi/chi — minimal, idiomatic router that is "just net/http." Use when you want composability with the standard library and stdlib middleware.
- valyala/fasthttp — the engine Fiber sits on. Use directly when you want the raw performance without the framework layer.
- Standard library net/http (Go 1.22+ routing) — Use when your routing needs are modest and you want zero dependencies plus HTTP/2 support.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0 | 2020-02 | Initial release; Express-style API over fasthttp[^1]. |
| v2.0 | 2020-09 | Major API revision; the long-lived stable line for years[^6]. |
| v3.0 | 2025 | Breaking restructure of context/middleware APIs and defaults[^5]. |

## References

[^1]: Fiber README and documentation — Express-inspired API design. https://github.com/gofiber/fiber
[^2]: valyala/fasthttp — underlying HTTP engine; documents lack of HTTP/2 support. https://github.com/valyala/fasthttp
[^3]: Fiber docs, "Ctx" — notes on the lifetime of returned values and immutability. https://docs.gofiber.io/api/ctx
[^4]: Fiber `adaptor` middleware — net/http bridge. https://docs.gofiber.io/api/middleware/adaptor
[^5]: Fiber v3 migration guide. https://docs.gofiber.io/next/guide/migration
[^6]: Fiber v2 release. https://github.com/gofiber/fiber/releases

## Tags

go, golang, web-framework, http, rest-api, fasthttp, express-inspired, backend, api, routing
