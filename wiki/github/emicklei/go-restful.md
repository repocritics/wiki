# emicklei/go-restful

> Resource-oriented REST framework for Go — and the routing layer inside the Kubernetes API server.

[GitHub repo](https://github.com/emicklei/go-restful) ·
[License: MIT](https://github.com/emicklei/go-restful/blob/v3/LICENSE)

## Overview

go-restful is one of the oldest surviving Go web frameworks: Ernest Micklei started it in November 2012, months after Go 1.0 shipped[^1]. Its design is openly modeled on Java's JAX-RS (JSR-311)[^2]: instead of registering bare handler functions on a mux, you declare `WebService` objects with routes, typed path/query parameters, consumed and produced MIME types, and documentation strings. That declarative metadata is the point — it is what allows OpenAPI documents to be generated from the code, and it is why the Kubernetes API server adopted go-restful for its REST endpoints and still depends on it today[^3].

The defining tension: go-restful's document-oriented, builder-style API predates almost every Go idiom that came later — `context.Context`, radix-tree routers, middleware as `func(http.Handler) http.Handler`. It wraps requests and responses in its own `restful.Request` / `restful.Response` types and uses a three-level filter chain instead of standard middleware. For greenfield Go services in 2026 this feels foreign; for API servers that need machine-readable route metadata it remains a deliberate choice.

At ~5,100 stars the project is modest by framework standards, but its installed base is enormous through Kubernetes. It is a single-maintainer project in a mature, low-churn state: 2 open issues, latest feature release v3.13.0 (2025-08), commits as recent as July 2026.

## Getting Started

```bash
go get github.com/emicklei/go-restful/v3
```

Note the `/v3` suffix — versions up to v2 (on `master`) predate Go modules; all module-aware development happens on the `v3` branch[^4].

```go
package main

import (
	"log"
	"net/http"

	restful "github.com/emicklei/go-restful/v3"
)

func findUser(req *restful.Request, resp *restful.Response) {
	id := req.PathParameter("user-id")
	resp.WriteAsJson(map[string]string{"id": id})
}

func main() {
	ws := new(restful.WebService)
	ws.Path("/users").
		Consumes(restful.MIME_JSON).
		Produces(restful.MIME_JSON)

	ws.Route(ws.GET("/{user-id}").To(findUser).
		Doc("get a user").
		Param(ws.PathParameter("user-id", "identifier of the user").DataType("string")))

	restful.Add(ws) // registers on the DefaultContainer
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Architecture / How It Works

Three nouns carry the design:

- **Container** — an `http.Handler` that owns a set of WebServices and dispatches requests to them. `restful.DefaultContainer` piggybacks on `http.DefaultServeMux`; multiple containers can serve different endpoints.
- **WebService** — a root path plus defaults (Consumes/Produces) and a list of routes.
- **Route** — built with a fluent builder: HTTP method, path template, handler (`RouteFunction`), typed parameters, `Reads(...)`/`Writes(...)` struct declarations, and doc strings. None of this metadata affects dispatch; it exists so tooling can read it.

**Routing.** Two interchangeable router implementations: the default `CurlyRouter` (static elements, `{var}` parameters, regexp segments like `{subpath:*}`, and Google-style custom verbs `/name:customVerb`) and `RouterJSR311`, which follows the JSR-311 matching algorithm. Neither is a radix tree; regexp evaluation is cached, with `SetPathTokenCacheEnabled` / `SetCustomVerbCacheEnabled` toggles to disable caching[^4].

**Filters, not middleware.** Cross-cutting logic is a `func(*Request, *Response, *FilterChain)` chain, attachable at container, WebService, or route level. Bundled filters cover CORS, automatic OPTIONS responses, and gzip/deflate content encoding. Plain `http.Handler` middleware can be adapted in via `HttpMiddlewareHandlerToFilter`.

**OpenAPI.** The framework itself generates nothing; the companion `emicklei/go-restful-openapi` walks route metadata to emit OpenAPI v2 (Swagger) documents[^5]. Kubernetes uses its own `kube-openapi` machinery on top of the same metadata. OpenAPI v3 output is not part of this stack.

**Context.** The handler signature predates `context.Context`. The wrapped `*http.Request` is exposed as `req.Request`, so the idiom is `req.Request.Context()` — workable, but the wrapper types leak into every handler signature and test.

## Production Notes

- **The Kubernetes coupling cuts both ways.** kube-apiserver is the largest consumer[^3], which makes the library conservatively maintained and very unlikely to break compatibility — and equally unlikely to modernize its API. Treat it as stable infrastructure, not an evolving framework.
- **CVE-2022-1996** (CVSS 9.1): the bundled CORS filter matched `AllowedDomains` as regular expressions, so a crafted origin like `myallowed.domain.attacker.com` could satisfy an `allowed.domain` entry. Fixed in v3.8.0 (2022-06) by exact-string matching[^6]. If you use `CrossOriginResourceSharing` on anything older, upgrade; this CVE also forced dependency bumps across the Kubernetes ecosystem.
- **Import-path trap.** `github.com/emicklei/go-restful` (no `/v3`) resolves to the ancient pre-modules line. Linters won't warn you; check your `go.mod`.
- **Throughput.** Fine for API servers; measurably behind radix-tree routers (gin, echo, chi) on routing micro-benchmarks, and each request allocates wrapper objects. v3.13.0 improved regexp-route matching speed[^7], but if routing overhead is your bottleneck this is the wrong library.
- **Bus factor is 1.** Ernest Micklei has run the project solo for over a decade. Activity is steady, but a small backlog and slow feature cadence are the norm — acceptable for a finished library, worth noting for long-horizon bets.

## When to Use / When Not

**Use when:**
- You need OpenAPI/Swagger documents generated from route declarations rather than hand-maintained.
- You are building Kubernetes-adjacent components (aggregated API servers, extension apiservers) where go-restful is already in the dependency graph.
- You value route metadata (params, types, docs) as a first-class, introspectable artifact.

**Avoid when:**
- You want idiomatic 2026 Go: `context.Context`-first handlers and `http.Handler` middleware — chi or the stdlib get you there with less ceremony.
- Raw routing performance matters — gin/echo are faster.
- You need OpenAPI v3 generation — huma or oapi-codegen approaches cover this natively.

## Alternatives

- gin-gonic/gin — use instead for the mainstream high-throughput choice with the largest middleware ecosystem.
- labstack/echo — similar performance class to gin with a more structured API and built-in binding/validation.
- go-chi/chi — use when you want stdlib-idiomatic `http.Handler` composition with zero framework lock-in.
- danielgtaylor/huma — use when OpenAPI 3.1 generation from Go types is the primary requirement (go-restful's niche, modernized).
- gorilla/mux — the classic request router; no metadata or OpenAPI story.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-11 | First public release, JSR-311-inspired design[^1]. |
| v1.2 | 2015-11 | Pluggable representations, compressor pools. |
| v3.0.0 | 2020-01 | Go modules support; development moves to `v3` branch[^4]. |
| v3.8.0 | 2022-06 | CORS `AllowedDomains` exact matching — CVE-2022-1996 fix[^6]. |
| v2.16.0 | 2022-07 | Late maintenance release on the legacy pre-modules line. |
| v3.12.0 | 2024-03 | Incremental maintenance release. |
| v3.13.0 | 2025-08 | Speed improvements for regexp-based routing[^7]. |

## References

[^1]: Ernest Micklei, "go-restful, first working example" — 2012-11. http://ernestmicklei.com/2012/11/go-restful-first-working-example/
[^2]: Ernest Micklei, "go-restful API design" — 2012-11. http://ernestmicklei.com/2012/11/go-restful-api-design/
[^3]: kubernetes/kubernetes `go.mod` — `github.com/emicklei/go-restful/v3` dependency. https://github.com/kubernetes/kubernetes/blob/master/go.mod
[^4]: go-restful README (v3 branch). https://github.com/emicklei/go-restful/blob/v3/README.md
[^5]: go-restful-openapi — OpenAPI v2 generation companion. https://github.com/emicklei/go-restful-openapi
[^6]: NVD, CVE-2022-1996 — go-restful CORS filter authorization bypass. https://nvd.nist.gov/vuln/detail/CVE-2022-1996
[^7]: go-restful v3.13.0 release notes. https://github.com/emicklei/go-restful/releases/tag/v3.13.0

## Tags

go, rest, web-framework, routing, openapi, swagger, kubernetes, http, api-server, middleware
