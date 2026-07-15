# go-chi/chi

> A lightweight HTTP router for Go that stays 100% `net/http`-compatible — handlers and middleware are just standard-library types.

[GitHub repo](https://github.com/go-chi/chi) ·
[Official website](https://go-chi.io) ·
[License: MIT](https://github.com/go-chi/chi/blob/master/LICENSE)

## Overview

chi is a router for building HTTP services in Go, written by Peter Kieltyka during the development of the Pressly API[^1]. It is built on Go's `context` package (introduced in Go 1.7) to carry request-scoped values, cancellation, and timeouts across a handler chain. The core router is small — roughly 1,000 lines — and ships with zero external dependencies beyond the standard library[^2].

Its defining decision is fidelity to `net/http`: a chi handler is a plain `http.HandlerFunc`, and a chi middleware is a plain `func(http.Handler) http.Handler`. There is no custom context object, no framework-specific handler signature, and no bespoke middleware type. This means any middleware in the wider Go ecosystem works unchanged, and code written against chi ports to and from the standard library with little friction. The tradeoff is that chi gives you a router and a middleware catalog — not a framework. Request binding, validation, rendering, and dependency injection are left to you or to sibling packages (`go-chi/render`, `go-chi/docgen`).

With ~22.5k stars and ~1.1k forks, chi is one of the most widely adopted Go routers, cited as being in production at Pressly, Cloudflare, Heroku, and others. Development has slowed to a maintenance cadence — the API has been stable for years — but the repository is still actively patched (last push 2026-07), most recently for security-sensitive client-IP handling (see Production Notes).

## Getting Started

```sh
go get -u github.com/go-chi/chi/v5
```

Note the `/v5` module suffix — it is required in both the `go get` and the import path.

```go
package main

import (
	"net/http"

	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
)

func main() {
	r := chi.NewRouter()
	r.Use(middleware.Logger)
	r.Use(middleware.Recoverer)

	r.Get("/users/{userID}", func(w http.ResponseWriter, r *http.Request) {
		id := chi.URLParam(r, "userID")
		w.Write([]byte("user " + id))
	})

	http.ListenAndServe(":3000", r)
}
```

URL patterns support named params (`/users/{userID}`), regex constraints (`/{slug:[a-z-]+}`), and a trailing wildcard (`/files/*`, read via `chi.URLParam(r, "*")`). Sub-routers compose with `r.Route(...)`, `r.Group(...)`, and `r.Mount(...)`.

## Architecture / How It Works

chi's router is a **Patricia radix trie**[^2]. Registered patterns are decomposed into a prefix tree; at request time chi walks the tree to find the matching route, extracting named parameters as it goes. Matching is allocation-conscious — parameter values are stored on a pooled `*chi.Context` retrieved from a `sync.Pool` and attached to the request context, so hot paths avoid per-request map allocation.

The public surface is the `chi.Router` interface, which embeds `http.Handler`. The concrete implementation is `*chi.Mux`. Because the mux is itself an `http.Handler`, you can nest muxes arbitrarily with `Mount`, and hand a chi router to anything expecting a standard handler (e.g. `http.Server`, `httptest.NewServer`, or another router).

Composition primitives:

- **`Use`** appends middleware to the current router's stack. Middleware must be registered before routes on that router — chi panics if you call `Use` after a route is already defined, because the trie is built eagerly.
- **`With`** attaches inline middleware for a single endpoint without mutating the parent stack.
- **`Group`** creates an inline sub-router sharing the same URL prefix but with its own middleware stack — used to apply auth to some routes but not their siblings.
- **`Route`** mounts a sub-router at a path prefix, the idiomatic way to structure a REST resource.
- **`Mount`** attaches an independent `http.Handler` (often a separately constructed mux) under `/prefix/*`.

The bundled `middleware` subpackage provides stdlib-only middlewares: `RequestID`, `Logger`, `Recoverer`, `Timeout`, `Throttle`, `Compress`, `BasicAuth`, `Heartbeat`, `Profiler`, and the client-IP family. None of these are privileged — they are ordinary `net/http` middleware you could have written yourself, which is the point.

## Production Notes

**`RealIP` is deprecated and unsafe.** The long-standing `middleware.RealIP` trusts `X-Forwarded-For` / `X-Real-IP` unconditionally and mutates `r.RemoteAddr`, making it spoofable when the server is reachable without a trusted proxy in front (advisories GHSA-3fxj-6jh8-hvhx and related). Recent releases add a `ClientIPFrom*` family — `ClientIPFromRemoteAddr`, `ClientIPFromHeader`, `ClientIPFromXFF`, `ClientIPFromXFFTrustedProxies` — where you must pick one matching your actual network topology and read the result via `GetClientIP`. If you rely on client IP for rate limiting or auth, audit this now.

**Middleware ordering is load-bearing and enforced by panic.** `Use` must precede route registration on a given (sub)router. A common mistake is calling `r.Use(...)` inside a `Route` closure after a `Get`, which panics at startup. Structure setup so all `Use` calls come first.

**Context-key collisions.** chi stores URL params under its own context key, but the README examples set user values with bare string keys (`context.WithValue(ctx, "user", ...)`), which vet flags and which can collide across packages. Use unexported typed keys in real code.

**Trailing-slash behavior is explicit, not automatic.** `/articles` and `/articles/` are distinct routes unless you opt into `middleware.StripSlashes` or `RedirectSlashes`. Mismatches surface as unexpected 404s.

**Standard-library overlap since Go 1.22.** Go 1.22 (2024) added method-and-wildcard patterns (`GET /users/{id}`) to the standard `http.ServeMux`. For simple services, stdlib routing may now be enough, narrowing chi's advantage to nested routers, per-group middleware stacks, regex params, and the middleware catalog. Evaluate whether you still need the dependency.

**Versioning.** Since v5 the module path carries the `/v5` suffix and go modules are required; pre-v5 import paths (`github.com/go-chi/chi` without the suffix) resolve to older, unmaintained code. New projects should always use `/v5`.

## When to Use / When Not

**Use when:**
- You want idiomatic `net/http` with better route composition than stdlib gives you.
- You are building a REST API that will grow — nested routers and per-group middleware keep large route tables maintainable.
- You want zero third-party dependencies and a small, auditable router.
- You need regex route params or middleware that any `net/http`-compatible package can satisfy.

**Avoid when:**
- Your routing needs are trivial and you are on Go 1.22+ — the standard `ServeMux` may suffice.
- You want an all-in-one framework with binding, validation, and rendering built in — reach for Gin, Echo, or Fiber.
- You need maximum raw routing throughput and are willing to trade `net/http` compatibility — httprouter/Gin's radix implementation and Fiber's fasthttp core benchmark faster.

## Alternatives

- gin-gonic/gin — batteries-included framework (binding, rendering, its own context); use instead when you want convenience over stdlib purity.
- labstack/echo — similar framework niche to Gin with a large middleware set; use when you prefer its API and want built-in features.
- gofiber/fiber — Express-style API built on fasthttp (not `net/http`); use when raw throughput matters more than stdlib compatibility.
- julienschmidt/httprouter — minimal high-performance radix router; use when you want the smallest, fastest primitive and will assemble the rest yourself.
- gorilla/mux — regex-first stdlib-compatible router; use when you need its matching features, noting Gorilla's maintenance has been intermittent.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-10 | Repository created; extracted from the Pressly API[^1]. |
| 1.0 | 2016 | First tagged release: radix-trie router on `net/http` + `context`. |
| 2.x–4.x | 2017–2019 | API refinement; middleware catalog growth. Pre-modules import path. |
| 5.0 | 2020-08 | Go modules support; import path gains the `/v5` suffix[^3]. |
| 5.x | 2021–2026 | Maintenance releases; deprecation of `RealIP` and addition of the `ClientIPFrom*` middleware family[^4]. |

## References

[^1]: chi README — origin at Pressly, design goals (stdlib-only, composability, ~1000 LOC core). https://github.com/go-chi/chi/blob/master/README.md
[^2]: chi README, "Router interface" — Patricia radix trie, full `net/http` compatibility, no external dependencies. https://github.com/go-chi/chi#router-interface
[^3]: chi CHANGELOG — v5 adds go.mod support and the `/v5` module path. https://github.com/go-chi/chi/blob/master/CHANGELOG.md
[^4]: chi middleware package — `RealIP` deprecation and `ClientIPFrom*` middlewares. https://pkg.go.dev/github.com/go-chi/chi/v5/middleware

## Tags

go, golang, http-router, net-http, rest-api, middleware, microservices, web-framework, backend, routing
