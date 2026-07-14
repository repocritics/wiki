# valyala/fasthttp

> A zero-allocation HTTP/1.x server and client for Go, built for high request-per-second workloads where net/http's per-request allocations dominate.

[GitHub repo](https://github.com/valyala/fasthttp) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/github.com/valyala/fasthttp) ·
[License: MIT](https://github.com/valyala/fasthttp/blob/master/LICENSE)

## Overview

fasthttp is an alternative HTTP implementation for Go, written by Aliaksandr Valialkin and public since 2015[^1]. Its single design goal is throughput: the hot request/response path performs zero heap allocations, achieved by pooling and reusing every object (`RequestCtx`, headers, argument buffers) across requests, and by exposing a byte-slice API (`[]byte`) instead of `string` so no copies are made on the read path. Benchmarks in the README put the server at up to ~6× and the client at up to ~4× the speed of the standard library on synthetic GET workloads[^2].

The project is unusually honest about its own scope. The README leads with a section titled "fasthttp might not be for you" and states that unless a service handles thousands of small-to-medium requests per second at low-millisecond latency, `net/http` is the better choice — easier to use, more complete, and fast enough that most teams will not notice the difference[^2]. fasthttp is a specialist tool, not a drop-in upgrade.

The defining tradeoff is that fasthttp is **not net/http-compatible and does not support HTTP/2**. It uses a `RequestHandler func(*RequestCtx)` signature instead of the `http.Handler` interface, ships no `ServeMux`, and speaks only HTTP/1.x. The performance comes from a memory-reuse model that leaks into the API contract in ways that cause real bugs (see Architecture). It is actively maintained — releases ship roughly monthly — and is best known today as the engine underneath the Fiber web framework.

## Getting Started

```bash
go get -u github.com/valyala/fasthttp
```

```go
package main

import (
	"fmt"

	"github.com/valyala/fasthttp"
)

func main() {
	handler := func(ctx *fasthttp.RequestCtx) {
		switch string(ctx.Path()) {
		case "/hello":
			fmt.Fprintf(ctx, "Hello, %s", ctx.QueryArgs().Peek("name"))
		default:
			ctx.Error("not found", fasthttp.StatusNotFound)
		}
	}

	// ListenAndServe blocks; the *RequestCtx carries both request and response.
	if err := fasthttp.ListenAndServe(":8080", handler); err != nil {
		panic(err)
	}
}
```

Note `string(ctx.Path())` — comparisons convert the byte slice at the call site rather than storing it. The handler is a plain function; there is no interface to implement and no `ResponseWriter`/`Request` pair.

## Architecture / How It Works

The central object is `RequestCtx`, which holds the parsed request, the response being built, and helper methods. One `RequestCtx` is allocated per connection worker and **reused for every request that worker serves**, drawn from a `sync.Pool`. This is where the allocation savings come from and also where the ecosystem's most common bug lives:

- **Everything returned by `ctx` is only valid until the handler returns.** `ctx.Path()`, `ctx.Request.Header.Peek(...)`, `ctx.FormValue(...)` and friends return byte slices into buffers the server will overwrite on the next request. Storing them in a map, closure, or goroutine that outlives the handler yields silent data corruption. To keep a value, copy it (`append([]byte(nil), b...)` or `string(b)`).
- **Response is buffered, not streamed by default.** Unlike net/http, fasthttp does not put bytes on the wire until the handler returns, so headers and body may be set, overwritten, and reordered freely during the handler. Streaming requires `ctx.SetBodyStream` / `Response.SetBodyStreamWriter`.
- **The server uses a worker-goroutine pool**, not one goroutine spawned per request from scratch, reducing scheduler and stack-growth overhead under high connection counts.

Data flow: `Server.Serve` accepts a connection, hands it to a pooled worker, which reads and parses requests into the reused `RequestCtx`, invokes the `RequestHandler`, flushes the response, and loops for keep-alive. Header parsing avoids allocation by scanning the raw buffer in place.

There is **no HTTP/2, no HTTP/3, and no automatic routing**. `ctx.Done()` does not behave like a standard `context.Context` per-request cancellation channel — it only closes on server shutdown, because allocating a channel per request would defeat the design[^2]. Routing is delegated to third-party libraries such as fasthttp/router, and higher-level ergonomics to frameworks like gofiber/fiber built on top.

## Production Notes

- **The retained-slice bug is the number-one footgun.** Any value derived from `ctx` that escapes the handler must be copied first. This surfaces as intermittent, load-dependent corruption that is hard to reproduce in tests. Code review for `fasthttp` handlers should treat every un-copied `ctx.*()` slice crossing a goroutine or cache boundary as a defect.
- **No HTTP/2 is a hard constraint, not a roadmap gap.** Browsers and gRPC clients expecting h2c or ALPN-negotiated HTTP/2 will not work. Terminate HTTP/2 at a reverse proxy (nginx, Envoy) and speak HTTP/1.1 to fasthttp behind it, or stay on net/http.
- **Multipart and body-size limits need explicit configuration.** Untrusted uploads should use `ctx.MultipartFormWithLimit()` or a `Server.MaxRequestBodySize` bound rather than the unbounded `ctx.MultipartForm()`, to avoid memory-exhaustion DoS.
- **The client is a different mental model from `net/http.Client`.** `fasthttp.Client` pools connections per host and reuses `Request`/`Response` objects; you are expected to `AcquireRequest`/`ReleaseRequest` and `AcquireResponse`/`ReleaseResponse`. Forgetting to release leaks pooled objects; using a released object races.
- **Ecosystem lock-in.** Standard middleware written against `http.Handler` does not compose. The `fasthttpadaptor` package bridges net/http handlers, but running through it forfeits most of the performance benefit and reintroduces allocations — at which point plain net/http is usually the better call.
- **Observability gaps.** Much of the Go tracing/metrics middleware ecosystem targets `http.Handler`; expect to write fasthttp-specific instrumentation or adopt a framework (Fiber) that provides it.

## When to Use / When Not

**Use when:**
- A proxy, edge node, or API gateway must sustain very high RPS with low, consistent tail latency and allocation pressure / GC pauses are the measured bottleneck.
- You control both ends or sit behind a proxy that handles TLS/HTTP2, so HTTP/1.x-only is acceptable.
- You are adopting Fiber and inherit fasthttp as its runtime.

**Avoid when:**
- You need HTTP/2, HTTP/3, or gRPC directly on the Go server.
- The team wants the standard-library ecosystem: `http.Handler` middleware, `context.Context` cancellation, ServeMux, tracing libraries.
- Throughput is not your proven bottleneck — the README itself recommends net/http for most services[^2].
- Handlers pass request-derived data to goroutines, caches, or long-lived structures where the copy-before-escape discipline is hard to enforce.

## Alternatives

- golang/go (net/http) — the standard library; HTTP/2 built in, full ecosystem, allocates per request. Use unless RPS/GC is a measured problem.
- gofiber/fiber — Express-style framework built on fasthttp; use when you want fasthttp speed with routing, middleware, and ergonomics included.
- gin-gonic/gin — popular net/http-based router/framework; use when you want net/http compatibility with better routing DX.
- fasthttp/router — thin high-performance router for raw fasthttp; use when you want fasthttp directly plus path routing, no framework.
- panjf2000/gnet — event-loop networking library for custom protocols; use when you need lower-level control than HTTP or non-HTTP TCP/UDP.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-10 | Repository made public; date-tagged snapshots (e.g. v20160316) precede semver[^1]. |
| v1.0.0 | 2018-09-13 | First semantic-version tag; API stabilized on `RequestCtx`[^3]. |
| v1.68.0 | 2025-10-23 | Ongoing monthly maintenance release. |
| v1.69.0 | 2026-01-05 | Maintenance release. |
| v1.70.0 | 2026-04-07 | Maintenance release. |
| v1.71.0 | 2026-05-05 | Maintenance release. |
| v1.72.0 | 2026-06-29 | Latest release at time of writing[^3]. |

## References

[^1]: valyala/fasthttp repository, created 2015-10-18. https://github.com/valyala/fasthttp
[^2]: fasthttp README — "fasthttp might not be for you", benchmark tables, and net/http conversion notes. https://github.com/valyala/fasthttp/blob/master/README.md
[^3]: fasthttp release tags on GitHub (v1.0.0 dated 2018-09-13; v1.72.0 dated 2026-06-29). https://github.com/valyala/fasthttp/tags

## Tags

go, http, http-server, http-client, networking, performance, zero-allocation, web-framework, http1, high-throughput
