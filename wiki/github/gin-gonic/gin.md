# gin-gonic/gin

> A Go HTTP web framework built on a radix-tree router, trading net/http idiom for a pooled context object and a large middleware ecosystem.

[GitHub repo](https://github.com/gin-gonic/gin) ·
[Official website](https://gin-gonic.com/) ·
[License: MIT](https://github.com/gin-gonic/gin/blob/master/LICENSE)

## Overview

Gin is one of the two or three defaults for building JSON HTTP APIs in Go, alongside the standard library, Echo, Fiber, and Chi. First released in 2014, its pitch has always been "a Martini-like API but much faster" — the README claims up to 40× the throughput of the (now-defunct) Martini framework, achieved by swapping reflection-based routing for a fork of `julienschmidt/httprouter`[^1]. In practice, Gin's differentiator is less raw speed than the ergonomics of `*gin.Context`: a single object that carries the request, response writer, path parameters, bound-and-validated input, per-request key/value state, and the middleware chain cursor.

The defining tension is that Gin is *not* idiomatic `net/http`. A Gin handler is `func(c *gin.Context)`, not `func(w http.ResponseWriter, r *http.Request)`. Middleware, error handling, and rendering all route through `gin.Context` rather than the standard interfaces. This buys terseness and a deep ecosystem, but it also means handlers, middleware, and test helpers written for Gin do not transfer to other frameworks, and vice versa. `gin.Engine` does implement `http.Handler`, so Gin composes at the *outer* boundary (you can mount it under any `net/http` server or wrap it), but everything inside the framework speaks Gin's dialect.

Gin is mature and widely deployed but slow-moving: the API has been stable for years, releases are infrequent, and the project is maintained conservatively by a small group of volunteers rather than a company. Treat it as settled infrastructure, not a fast-evolving framework.

## Getting Started

Gin requires Go 1.25 or newer as of the current release[^2] — an unusually aggressive minimum for a library this widely used, so pin your toolchain before adopting.

```bash
go get github.com/gin-gonic/gin
```

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type CreateUser struct {
	Name  string `json:"name" binding:"required"`
	Email string `json:"email" binding:"required,email"`
}

func main() {
	r := gin.Default() // Logger + Recovery middleware attached

	r.POST("/users", func(c *gin.Context) {
		var in CreateUser
		if err := c.ShouldBindJSON(&in); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusCreated, gin.H{"name": in.Name})
	})

	r.Run(":8080") // shorthand for http.ListenAndServe
}
```

## Architecture / How It Works

**Router.** Gin's routing tree is a modified fork of httprouter[^1]: a per-HTTP-method radix (prefix) tree of path segments. Lookups are allocation-free, which is the source of the `0 allocs/op` figures in the README benchmark table. The tradeoff is that route registration is strict — the tree cannot hold two patterns that are structurally ambiguous at the same position, and Gin *panics at startup* when it detects a conflict (e.g. mixing a wildcard `:id` and a static segment under the same parent in incompatible ways). This is a compile-of-routes-at-boot model, not a first-match-wins regex list like `gorilla/mux`.

**Context and pooling.** `*gin.Context` is recycled through a `sync.Pool`. On each request the engine pulls a context, resets it, runs the handler chain, and returns it to the pool. This is why the single most common Gin bug is retaining a `*gin.Context` past the request — passing it into a goroutine that outlives the handler reads a context that has already been reset and reused for another request. The framework provides `c.Copy()` specifically for this case.

**Middleware chain.** Middleware and route handlers are the same type, `gin.HandlerFunc`, stored in a flat slice on the context. `c.Next()` advances an integer index and invokes the next handler; a middleware runs code before `c.Next()` (pre-processing) and after it returns (post-processing), giving the familiar onion model. `c.Abort()` sets the index past the end so no further handlers run — but note that `Abort` does **not** return from the current function; you must `return` yourself after calling it. `gin.Default()` is just `gin.New()` plus the `Logger()` and `Recovery()` middleware.

**Binding and validation.** `c.ShouldBind*` inspects content type (or an explicit binder) to unmarshal JSON, XML, form, query, URI params, or headers into a struct, then runs struct-tag validation via `go-playground/validator/v10`[^3]. There are two families: `ShouldBind…` returns an error for you to handle, while `Bind…` writes a 400 and aborts automatically — a subtle behavioral fork that trips up newcomers who expect `Bind` to be a pure decoder.

**Rendering.** A `render` package abstracts output (JSON, IndentedJSON, XML, YAML, ProtoBuf, HTML templates, string, redirect, file). `gin.H` is a convenience alias for `map[string]any`.

## Production Notes

**The context-in-goroutine footgun** is the top real-world failure. Any background work (logging to an external service, fan-out fetches) must operate on `c.Copy()` or on values extracted before the goroutine starts. Symptoms are nondeterministic — wrong user data, nil panics, races — because they depend on pool reuse timing under load.

**Panic recovery hides nothing about correctness.** `Recovery()` converts a panicking handler into a 500 and keeps the server alive, which is good for uptime but means a latent nil-deref or index-out-of-range can sit in production returning 500s without crashing. Pair it with real error monitoring; don't treat "the server didn't fall over" as "the code is fine."

**Validation error messages are poor by default.** `validator/v10` returns errors like `Key: 'CreateUser.Email' Error:Field validation for 'Email' failed on the 'email' tag`. Surfacing these raw to API clients leaks struct field names and reads badly; most production codebases write a translation layer or use the validator's translation registry.

**Security history.** Gin has shipped CVEs — notably a file-name/`Content-Disposition` handling issue fixed in 1.9.1[^4], and earlier a `Content-Type` header injection. None were architectural, but they underline that pinning to a current patch release matters; the infrequent release cadence means fixes can lag disclosure.

**Trailing-slash redirects and route conflicts** surprise people migrating from mux-style routers: Gin redirects `/foo/` to `/foo` (configurable via `RedirectTrailingSlash`), and it will panic at boot on an ambiguous route rather than resolving it at request time. Both are startup-visible, so they fail fast in tests rather than silently in prod.

**Not a batteries-included server.** Gin is a router + middleware harness. There is no built-in graceful shutdown (use `http.Server.Shutdown` around `engine.Handler()`), no ORM, no config, no dependency injection. This is by design and keeps the surface small, but greenfield teams should budget for assembling the surrounding stack.

## When to Use / When Not

**Use when:**
- You're building JSON REST APIs or microservices and want terse handlers plus request binding/validation out of the box.
- You want a large, battle-tested middleware ecosystem (`gin-contrib`: CORS, JWT, sessions, gzip, prometheus, zap logging).
- You value a stable, rarely-breaking API over cutting-edge features.

**Avoid when:**
- You want to stay in idiomatic `net/http` so handlers/middleware stay portable — reach for Chi or the standard library's `http.ServeMux` (which gained method + wildcard routing in Go 1.22).
- You need the absolute lowest latency/allocations and are willing to leave `net/http` entirely — Fiber (on fasthttp) benchmarks higher, at the cost of `net/http` incompatibility.
- Your service is small enough that a few standard-library handlers avoid a framework dependency altogether.

## Alternatives

- labstack/echo — closest peer; similar context/middleware model, comparable performance, arguably cleaner binding. Use instead when you prefer Echo's API and error-handling ergonomics.
- go-chi/chi — stays on stdlib `net/http` types (`http.Handler`, `http.HandlerFunc`). Use instead when handler portability and idiomatic Go matter more than a bundled context object.
- gofiber/fiber — built on `valyala/fasthttp`, not `net/http`. Use instead when you want maximum throughput and can accept incompatibility with the standard HTTP ecosystem.
- gorilla/mux — regex/first-match router, no framework opinions. Use instead when you only need flexible routing and want to assemble everything else yourself.
- Go standard library `net/http` — since Go 1.22 the default `ServeMux` supports method-based and wildcard patterns. Use instead when your routing needs are modest and you want zero framework dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014–2017 | Initial development; httprouter-based radix router established. |
| 1.0 | 2017 | First stable tag; API largely frozen after this. |
| 1.6.0 | 2020 | Binding improvements, `context` propagation refinements. |
| 1.7.x | 2021 | Bug fixes, expanded binders and render types. |
| 1.9.1 | 2023 | Security release fixing a `Content-Disposition` file-name CVE[^4]. |
| 1.10.0 | 2024 | Maintenance and dependency updates. |
| 1.12.0 | 2026 | Current release; Go 1.25 minimum, new features and fixes[^2]. |

## References

[^1]: Gin README — "a Martini-like API but with significantly better performance—up to 40 times faster—thanks to httprouter." https://github.com/gin-gonic/gin
[^2]: Gin 1.12.0 release announcement (Go 1.25 minimum). https://gin-gonic.com/en/blog/news/gin-1-12-0-release-announcement/
[^3]: go-playground/validator — struct validation used by Gin's binding layer. https://github.com/go-playground/validator
[^4]: CVE-2023-29401 — improper handling of filename in `Content-Disposition`, fixed in Gin 1.9.1. https://github.com/gin-gonic/gin/security/advisories

## Tags

go, http, web-framework, rest-api, router, middleware, microservices, backend, radix-tree, json-api
