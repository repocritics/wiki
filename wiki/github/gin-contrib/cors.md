# gin-contrib/cors

> CORS middleware for the Gin web framework — a thin per-request header setter and preflight responder driven by one config struct.

[GitHub repo](https://github.com/gin-contrib/cors) ·
[Gin website](https://gin-gonic.github.io/gin/) ·
[License: MIT](https://github.com/gin-contrib/cors/blob/master/LICENSE)

## Overview

`gin-contrib/cors` is the official Cross-Origin Resource Sharing middleware for [Gin](https://github.com/gin-gonic/gin), maintained under the same `gin-contrib` org that houses Gin's other first-party middleware. It does one job: inspect the incoming `Origin` header, decide whether the request is allowed, write the `Access-Control-Allow-*` response headers, and short-circuit browser preflight (`OPTIONS`) requests. It is not a general-purpose security layer — CORS is a browser-enforced policy, and this middleware only produces the headers that instruct compliant browsers what cross-origin JavaScript may read.

The library is small, stable, and heavily depended upon: it is one of the most-used pieces in the Gin ecosystem and changes rarely. Its defining tension is that CORS is easy to configure wrongly in ways that either silently break frontends or quietly widen access. The middleware faithfully implements the spec's hard rules — most notably that a wildcard origin (`AllowAllOrigins`) is incompatible with credentialed requests (cookies, `Authorization`) — and it will panic or error at construction on contradictory configs rather than fail open at runtime. That strictness is a feature, but it surprises developers who expect `cors.Default()` plus `AllowCredentials: true` to "just work."

The mental model to keep: this is middleware you register once with `router.Use(...)`, and it runs on every request. It does not do authentication, rate limiting, or CSRF protection, and CORS is not a substitute for any of them.

## Getting Started

```sh
go get github.com/gin-contrib/cors
```

```go
package main

import (
	"time"

	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.Use(cors.New(cors.Config{
		AllowOrigins:     []string{"https://app.example.com"},
		AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
		AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
		ExposeHeaders:    []string{"Content-Length"},
		AllowCredentials: true, // valid only because AllowOrigins is explicit, not "*"
		MaxAge:           12 * time.Hour,
	}))

	r.GET("/ping", func(c *gin.Context) { c.JSON(200, gin.H{"pong": true}) })
	r.Run(":8080")
}
```

`cors.Default()` is a one-liner that allows all origins — convenient for public read-only APIs, but it forces `AllowCredentials` off and cannot be combined with cookies.

## Architecture / How It Works

`cors.New(config)` calls `config.Validate()` once, at startup, and returns a `gin.HandlerFunc`. Validation is where contradictory configs are caught eagerly: setting `AllowAllOrigins` alongside `AllowOrigins` or an origin function is an error, a wildcard in an origin string without `AllowWildcard` panics, and a config that specifies no origin policy at all is rejected. Whatever normalization is possible (splitting schemes, precomputing the joined header values for methods and headers) is done at build time so the per-request path stays cheap.

At request time the handler reads the `Origin` header. If it is empty (a same-origin or non-browser request), the middleware calls `c.Next()` and gets out of the way. If an origin is present, it is checked against the configured policy in this order of precedence: `AllowOriginWithContextFunc` (gets the full `*gin.Context`), then `AllowOriginFunc`, then the static `AllowOrigins` list / `AllowAllOrigins`. A disallowed origin causes the request to be aborted with `403`.

For allowed origins the handler splits on HTTP method. A preflight `OPTIONS` request is answered directly: it writes `Access-Control-Allow-Methods`, `-Allow-Headers`, `-Max-Age`, and (when configured) the private-network header, then aborts with the configured status (`204` by default) — the route handler never runs. For a normal request it writes `Access-Control-Allow-Origin`, `-Expose-Headers`, and the credentials/vary headers, then calls `c.Next()` so the actual handler executes.

Origin matching beyond exact string equality is opt-in. `AllowWildcard` enables a single `*` per origin (e.g. `https://*.example.com`); non-HTTP schemes require explicit flags — `AllowBrowserExtensions` for `chrome-extension://`, `AllowWebSockets` for `ws://`/`wss://`, `AllowFiles` for `file://`, and `CustomSchemas` for things like `tauri://`. These exist because CORS in desktop/extension contexts (Tauri, Electron, browser extensions) sends non-web origins that the strict validator would otherwise reject.

## Production Notes

- **Wildcard origin plus credentials is not allowed, by spec.** `AllowAllOrigins: true` (or `cors.Default()`) makes the middleware refuse to reflect cookies; browsers reject `Access-Control-Allow-Origin: *` on credentialed requests anyway. If your frontend sends cookies or `Authorization`, you must enumerate exact origins or use `AllowOriginFunc`. This is the single most common support issue.
- **Reflecting the Origin header is a footgun.** A naive `AllowOriginFunc` that returns `true` for everything, combined with `AllowCredentials`, effectively echoes any origin back and defeats CORS entirely. Validate against a real allowlist.
- **`AllowHeaders` must include what the client actually sends.** Browsers list non-simple request headers in the preflight `Access-Control-Request-Headers`; anything not in your `AllowHeaders` (commonly `Authorization`, `Content-Type` with non-form values, custom `X-` headers) makes the preflight fail and the real request never fires. Failures show up in the browser console, not in Gin logs.
- **Register order matters.** `router.Use(cors.New(...))` should come before route groups and before middleware that can abort early (auth). If an auth middleware rejects the preflight `OPTIONS` before CORS runs, the browser sees a failed preflight rather than a CORS decision.
- **`MaxAge` is capped by browsers.** Setting a large `MaxAge` does not guarantee long preflight caching — Chromium caps preflight cache at 2 hours and Firefox at 24 hours regardless of the header value.
- **Not a security boundary.** CORS only constrains browser-based cross-origin reads. It does nothing against `curl`, server-to-server calls, or CSRF via simple requests. Do not treat an origin allowlist as authentication.
- **`AllowPrivateNetwork` targets a still-shifting spec.** The Private Network Access header is emitted for you, but the browser-side behavior around it has changed over Chromium versions; verify against the browsers you actually support.

## When to Use / When Not

**Use when:**
- You run a Gin HTTP API consumed by a browser frontend on a different origin.
- You want spec-correct preflight handling without writing header logic by hand.
- You need per-request origin decisions (multi-tenant subdomains, dynamic allowlists) via `AllowOriginFunc` / `AllowOriginWithContextFunc`.

**Avoid / reconsider when:**
- Your API is server-to-server only — CORS headers are irrelevant and add noise.
- You need CSRF protection, auth, or rate limiting — those are separate middleware; CORS solves none of them.
- You are not on Gin — the package is bound to `gin.HandlerFunc` and `*gin.Context`; use `rs/cors` for stdlib `net/http` or other frameworks.

## Alternatives

- rs/cors — framework-agnostic `net/http` CORS handler; use when you are not on Gin or want one CORS implementation across multiple Go frameworks.
- go-chi/cors — the chi router's port of `rs/cors`; use when your stack is chi rather than Gin.
- gorilla/handlers — includes `CORS()` alongside logging/compression handlers; use when you already depend on gorilla and want CORS bundled in.
- labstack/echo (middleware.CORS) — Echo's built-in CORS; use when you chose Echo instead of Gin.
- Hand-rolled middleware — for a single fixed origin with no preflight complexity, a few lines setting `Access-Control-Allow-Origin` may be simpler than a dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.1 | 2016-11-09 | Early tagged release alongside Gin's middleware split-out. |
| v1.3.0 | 2019-05-09 | Config/API consolidation during Go-modules era. |
| v1.4.0 | 2022-07-05 | Wildcard and custom-scheme handling maturing. |
| v1.5.0 | 2023-11-25 | Context-aware origin function (`AllowOriginWithContextFunc`) line of work. |
| v1.6.0 | 2024-03-06 | Configuration and validation refinements. |
| v1.7.0 | 2024-03-10 | Private Network Access / options-status and related config additions. |
| v1.7.6 | 2025-06-20 | Maintenance and dependency bumps. |
| v1.7.7 | 2026-03-28 | Latest release; small maintenance cadence. |

Roughly one to two tagged releases a year in recent history — a maintenance-mode library whose surface is intentionally stable. Last push to `master` was 2026-06-26.

## References

[^1]: gin-contrib/cors README and configuration reference. https://github.com/gin-contrib/cors
[^2]: Package GoDoc — `cors.Config` fields and helper methods. https://pkg.go.dev/github.com/gin-contrib/cors
[^3]: Gin web framework. https://github.com/gin-gonic/gin
[^4]: MDN — Cross-Origin Resource Sharing (CORS), including the wildcard-plus-credentials rule. https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
[^5]: rs/cors — framework-agnostic Go CORS handler. https://github.com/rs/cors

## Tags

go, golang, gin, middleware, cors, http, web-security, api, cross-origin, backend
