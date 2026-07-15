# rs/cors

> A `net/http`-compatible CORS middleware for Go — one small package that implements the Cross-Origin Resource Sharing protocol correctly, with zero per-request allocations.

[GitHub repo](https://github.com/rs/cors) ·
[License: MIT](https://github.com/rs/cors/blob/master/LICENSE)

## Overview

`rs/cors` is a single-purpose Go library that answers cross-origin preflight (`OPTIONS`) requests and attaches the correct `Access-Control-*` response headers to actual requests. It is written by Olivier Poitrey (`rs`), also the author of `zerolog` and `xid`, and has been the de facto standalone CORS handler for the Go `net/http` ecosystem since 2014[^1]. It is not a framework; it is a `http.Handler` decorator plus adapters for the common router/middleware libraries (Gin, Chi, Negroni, Gorilla, Buffalo, Alice, HttpRouter, and others), each shipped as an example rather than a hard dependency[^2].

The library exists because CORS is deceptively easy to get subtly wrong. The spec distinguishes simple requests from preflighted ones, requires precise `Vary` headers for cache correctness, and has a hard browser rule that `Access-Control-Allow-Origin: *` cannot be combined with credentials. `rs/cors` encodes those rules once so application code does not re-derive them. Its defining tradeoff is scope discipline: it does exactly CORS and nothing adjacent (no auth, no CSRF, no rate limiting), and it deliberately refuses some insecure-but-convenient configurations that ad-hoc implementations allow.

Many teams reach for it only after hand-rolled `w.Header().Set("Access-Control-Allow-Origin", "*")` code fails a preflight or leaks credentials. The package's value is that it is boring, small, and correct.

## Getting Started

```bash
go get github.com/rs/cors
```

```go
package main

import (
	"net/http"

	"github.com/rs/cors"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"hello":"world"}`))
	})

	// Explicit, credential-safe configuration.
	c := cors.New(cors.Options{
		AllowedOrigins:   []string{"https://app.example.com", "https://*.example.com"},
		AllowedMethods:   []string{http.MethodGet, http.MethodPost, http.MethodPut},
		AllowedHeaders:   []string{"Authorization", "Content-Type"},
		AllowCredentials: true,
		MaxAge:           300, // seconds; browsers cache the preflight
	})

	http.ListenAndServe(":8080", c.Handler(mux))
}
```

`cors.Default()` is a shortcut that allows all origins with simple methods (`GET`, `POST`) and no credentials — fine for public read APIs, wrong for anything with cookies or auth headers.

## Architecture / How It Works

The library compiles an `Options` struct into an immutable `*Cors` value once, at construction. All per-request work is lookups against pre-normalized data, which is why the benchmarks report `0 allocs/op`[^2]:

- Allowed methods and headers are upper/canonical-cased up front so the request path only compares.
- Wildcard origins (`https://*.example.com`) are split into prefix/suffix matchers at build time rather than compiled to regex per request. Wildcards carry a small documented match cost versus exact-string origins.

At request time `Cors.Handler` wraps the next `http.Handler` and branches on request shape:

1. **Preflight** — `OPTIONS` carrying an `Access-Control-Request-Method` header. The middleware validates origin, method, and requested headers, writes the `Access-Control-Allow-*` response, sets the appropriate `Vary` headers, and (by default) terminates the request with `204 No Content`. If `OptionsPassthrough` is set, it instead calls the next handler.
2. **Actual request** — origin is validated, `Access-Control-Allow-Origin` (plus `-Credentials` / `-Expose-Headers`) is set, and the request is always passed through to the next handler. The middleware never short-circuits a non-preflight request; a disallowed origin simply gets no CORS headers and the browser enforces the block.

Origin validation has a layered override chain, newest wins: `AllowOriginVaryRequestFunc(r, origin) (bool, []string)` (returns extra `Vary` headers) supersedes `AllowOriginRequestFunc` (deprecated) which supersedes `AllowOriginFunc` which supersedes the static `AllowedOrigins` list. Setting a function silently ignores the lists below it — a common source of "my AllowedOrigins is being ignored" confusion.

`Vary` handling is a first-class concern: the library adds `Vary: Origin` (and on preflight `Access-Control-Request-Method` / `-Headers`) so shared caches and CDNs do not serve one origin's CORS response to another.

## Production Notes

**The `*` + credentials footgun.** Browsers reject `Access-Control-Allow-Origin: *` when the response also allows credentials. Earlier versions of this library worked around that by reflecting the request `Origin` back — effectively "allow any origin with cookies", a serious vulnerability. That reflection was removed in issues [#55]/[#57][^3]. Consequence today: `AllowedOrigins: ["*"]` + `AllowCredentials: true` does **not** silently do what naive code expects. To allow any origin *with* credentials you must opt in explicitly via `AllowOriginFunc(func(string) bool { return true })` and accept the risk. This is the single most important behavioral caveat when upgrading from a hand-rolled handler.

**`MaxAge` defaults to 0**, meaning "do not cache the preflight" — every credentialed or non-simple request pays a second round trip. Set it (e.g. 300–600s) in latency-sensitive APIs. Note browsers cap it regardless of the value you send (Chromium caps at 2 hours), so a huge `MaxAge` does not eliminate re-preflighting.

**`OptionsPassthrough` is easy to misuse.** Turn it on only if a downstream handler genuinely serves `OPTIONS`; otherwise preflights fall through to routes that were not written to answer them, producing 404/405 responses that browsers read as CORS failures.

**Header/method matching is exact.** Non-listed request headers cause preflight rejection. Auth stacks that add `Authorization`, `X-Requested-With`, or custom headers must enumerate them (or the request will pass locally with tools like curl but fail in a real browser).

**`AllowPrivateNetwork`** targets the Private Network Access draft (`Access-Control-Request-Private-Network`); it is only meaningful for the subset of Chromium builds that implement that evolving spec — do not rely on it as a portable security boundary.

**Caching correctness.** If you put a CDN in front of a varying-origin API, verify the cache honors `Vary: Origin`; otherwise the CDN can cache the first origin's `Access-Control-Allow-Origin` and serve it to everyone, breaking CORS for the rest. The library sets `Vary` correctly — the failure mode is downstream cache configuration.

**Debug mode** (`Debug: true`) logs the middleware's decision path. It is per-request logging on the hot path; leave it off in production.

## When to Use / When Not

**Use when:**
- You have a Go `net/http` (or Gin/Chi/Negroni/Gorilla/etc.) service that a browser front-end on another origin calls.
- You want CORS decisions in one audited place instead of scattered `Header().Set` calls.
- You need dynamic origin allow-listing (per-tenant, per-request) via `AllowOriginVaryRequestFunc`.

**Avoid / unnecessary when:**
- Your API and front-end are same-origin (or served through a reverse proxy under one origin) — CORS never triggers, so the middleware is dead weight.
- You want CORS handled at the edge — API gateways, Envoy, nginx, or a CDN can enforce it before Go, offloading preflights entirely.
- You expected an auth/CSRF layer: this is neither; pair it with something else.

## Alternatives

- gin-contrib/cors — use instead when you are already all-in on Gin and want the config expressed in Gin's idioms.
- go-chi/cors — a Chi-maintained port of this same library; use when you want zero external dependencies inside a Chi stack.
- gorilla/handlers — its `CORS` handler covers the basics; use when you already depend on Gorilla and do not need wildcard-origin or dynamic-origin features.
- Handle CORS at nginx / Envoy / your API gateway — use instead when you want preflights answered before they ever reach Go, or want one policy across many backend languages.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0 | 2016-06-17 | First tagged release; core `net/http` handler[^1]. |
| v1.6.0 | 2018-10-01 | Continued `Options`/matching refinements. |
| v1.7.0 | 2019-07-09 | `AllowOriginRequestFunc` request-aware origin validation. |
| v1.8.0 | 2021-06-07 | `AllowPrivateNetwork` (Private Network Access support). |
| v1.9.0 | 2023-03-01 | Wildcard-origin and `Vary`-handling improvements. |
| v1.10.0 | 2023-09-05 | `AllowOriginVaryRequestFunc` returning extra `Vary` headers. |
| v1.11.0 | 2024-04-24 | Further `Vary` / preflight correctness work. |
| v1.11.1 | 2024-08-29 | Latest tag as of this writing; maintenance fix. |

The project is mature and low-churn: last pushed 2026-06-04, ~2.9k stars, ~230 forks, MIT-licensed, not archived[^4]. Development is maintenance-mode — infrequent commits reflecting a stable, feature-complete scope rather than abandonment.

## References

[^1]: `rs/cors` repository and README. https://github.com/rs/cors
[^2]: README — Parameters, framework examples, and zero-allocation benchmarks. https://github.com/rs/cors/blob/master/README.md
[^3]: Security fix removing `Origin` reflection under `*` + credentials. https://github.com/rs/cors/issues/55 and https://github.com/rs/cors/issues/57
[^4]: GitHub repository metadata (stars, forks, license, last push), fetched 2026-07-15. https://github.com/rs/cors

## Tags

go, cors, http-middleware, net-http, web-security, preflight, cross-origin, api, backend, http-handler
