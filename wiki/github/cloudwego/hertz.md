# cloudwego/hertz

> ByteDance's Go HTTP framework — a fasthttp fork with gin-style ergonomics, built on the Netpoll network library.

[GitHub repo](https://github.com/cloudwego/hertz) ·
[Official website](https://www.cloudwego.io) ·
[License: Apache-2.0](https://github.com/cloudwego/hertz/blob/main/LICENSE)

## Overview

Hertz is an HTTP server framework for Go, open-sourced by ByteDance in June 2022 as part of the CloudWeGo suite (alongside the Kitex RPC framework and the Netpoll network library)[^1]. It began as a fork of `valyala/fasthttp` and borrows its routing and middleware ergonomics from gin and echo, wrapped around ByteDance's internal networking stack[^2]. It is used in production across ByteDance's microservices, which is the primary reason to take a still-pre-1.0 framework seriously.

The defining design choice is the **pluggable transport layer**. By default Hertz runs on Netpoll, an epoll/kqueue-based, non-goroutine-per-connection network library also maintained by CloudWeGo, and it can switch to Go's standard `net` library at construction time. This is the framework's main differentiator from gin (which is `net/http`-based) and its main source of platform-specific caveats — Netpoll does not run everywhere the standard library does.

The framework is layered (application / protocol / transport), heavily interface-driven, and ships a code-generation tool (`hz`) that scaffolds services from Thrift or Protobuf IDL. Its center of gravity is Chinese-speaking: much of the deepest documentation, issue discussion, and design blogs are primarily in Chinese, with English translations that sometimes lag[^3].

## Getting Started

```bash
go get github.com/cloudwego/hertz@latest
```

```go
package main

import (
	"context"

	"github.com/cloudwego/hertz/pkg/app"
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/protocol/consts"
)

func main() {
	h := server.Default(server.WithHostPorts(":8080"))

	// Handlers take BOTH a context.Context and a pooled *app.RequestContext.
	h.GET("/ping", func(ctx context.Context, c *app.RequestContext) {
		c.JSON(consts.StatusOK, map[string]string{"message": "pong"})
	})

	h.Spin() // blocks; installs graceful-shutdown signal handling
}
```

Optionally, install the `hz` code generator to scaffold from IDL:

```bash
go install github.com/cloudwego/hertz/cmd/hz@latest
hz new -idl ./idl/api.thrift
```

## Architecture / How It Works

Hertz is built in three layers, each an interface with a default implementation you can replace:

1. **Transport** — Netpoll by default, or Go `net`. Netpoll uses an event-loop / connection-multiplexing model instead of a goroutine per connection, and a zero-copy `LinkBuffer` for reads/writes. This is where Hertz's throughput claims come from under high connection counts[^2].
2. **Protocol** — HTTP/1.1 is native. ALPN negotiation is supported, but full HTTP/2 is not in core; it lives in `hertz-contrib/http2`[^4].
3. **Application** — a radix-tree router (the gin/httprouter lineage), middleware chain, data binding, and rendering.

The handler signature `func(ctx context.Context, c *app.RequestContext)` is deliberate: `ctx` is the standard request-scoped context for tracing and cancellation, while `*app.RequestContext` holds request/response state and is **pooled via `sync.Pool`** and reset between requests. Keeping the two separate lets Hertz reuse the heavy request object without breaking `context.Context` propagation.

The `hz` tool generates router registration, handler stubs, and model structs from Thrift/Protobuf IDL, which is how ByteDance keeps HTTP and Kitex RPC services consistent. Extensions (auth, sessions, CORS, gzip, OpenTelemetry, service registry/discovery for nacos/consul/etcd/etc.) live in the separate `hertz-contrib` organization rather than core[^5].

## Production Notes

**Netpoll is not universal.** Netpoll targets Linux (epoll) and macOS/BSD (kqueue); it does not support Windows, where Hertz falls back to the standard `net` library[^2]. Netpoll's advantage narrows or reverses in low-connection-count / low-concurrency scenarios — the official guidance is to benchmark your own workload and switch transports with `server.WithTransport(standard.NewTransporter)` if Go `net` wins.

**The pooled RequestContext is a footgun.** Because `*app.RequestContext` is reset and reused after the handler returns, passing it to another goroutine (or retaining it past `Spin`'s handler scope) reads freed/overwritten state. Use `c.Copy()` before handing context to a background goroutine. This is the same hazard gin has with `*gin.Context`, and it bites people migrating naive concurrent handlers.

**Body streaming is opt-in.** By default the request body is fully read into memory; large uploads or streaming request bodies require `server.WithStreamRequestBody(true)`. Getting this wrong shows up as memory pressure under big payloads.

**Still pre-1.0.** Despite heavy internal use, Hertz has never tagged a 1.0 release — the current line is v0.10.x (v0.10.5, June 2026)[^6]. The API is stable in practice, but the semver signal means you should pin versions and read release notes before minor upgrades.

**Ecosystem coupling.** Netpoll, Kitex, and the observability extensions are all CloudWeGo projects; adopting Hertz tends to pull in the CloudWeGo stack. That is a strength if you also run Kitex RPC and a friction point if you want a standalone HTTP library with a large neutral middleware market.

## When to Use / When Not

**Use when:**
- You run high-concurrency Go microservices and can benchmark a real Netpoll win.
- You already use CloudWeGo (Kitex) and want IDL-driven HTTP + RPC consistency.
- You want a layered, replaceable transport/protocol design rather than `net/http` lock-in.
- You need ByteDance-scale battle-testing more than a 1.0 label.

**Avoid when:**
- You target Windows in production (Netpoll falls back to standard `net`).
- You want the largest middleware ecosystem and English-first docs — gin's community is bigger.
- You need native HTTP/2 or HTTP/3 in core (HTTP/2 is a contrib add-on).
- Your service is low-traffic/low-concurrency, where gin or echo's maturity outweighs any perf delta.

## Alternatives

- gin-gonic/gin — the most popular Go web framework; `net/http`-based, larger community, use when ecosystem and familiarity beat raw throughput.
- labstack/echo — similar router/middleware ergonomics on `net/http`; use for a lighter, well-documented default.
- valyala/fasthttp — the low-level library Hertz forked; use when you want maximum control and no framework layer.
- gofiber/fiber — fasthttp-based, Express-style API; use if you prefer that DSL and accept fasthttp's non-`net/http` semantics.
- go-kratos/kratos — full microservice framework (HTTP + gRPC + tooling); use when you need more than an HTTP layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.0.1 | 2022-05-31 | First public tag ahead of announcement[^1]. |
| v0.1.0 | 2022-06-20 | Open-source launch release. |
| v0.3.0 | 2022-08-29 | Router and binding improvements. |
| v0.5.0 | 2023-01-09 | Adaptor and middleware maturation. |
| v0.6.0 | 2023-02-27 | Protocol-layer extensibility work. |
| v0.7.0 | 2023-09-26 | Standard `net` transport option hardened. |
| v0.8.0 | 2024-01-12 | Binding/validation and config refinements. |
| v0.9.0 | 2024-05-10 | Continued API and performance work. |
| v0.10.0 | 2025-05-15 | Latest minor line. |
| v0.10.5 | 2026-06-11 | Current patch release[^6]. |

## References

[^1]: CloudWeGo blog, "Ultra-large-scale Enterprise-level Microservice HTTP Framework — Hertz is Officially Open Source" — 2022-06-21. https://www.cloudwego.io/blog/2022/06/21/
[^2]: CloudWeGo, "ByteDance Practice on Go Network Library" (Netpoll design). https://www.cloudwego.io/blog/2020/05/24/bytedance-practices-on-go-network-library/
[^3]: Hertz documentation. https://www.cloudwego.io/docs/hertz/
[^4]: hertz-contrib/http2 — HTTP/2 support for Hertz. https://github.com/hertz-contrib/http2
[^5]: hertz-contrib — community extension organization. https://github.com/hertz-contrib
[^6]: Hertz releases. https://github.com/cloudwego/hertz/releases

## Tags

go, http, web-framework, microservices, netpoll, cloudwego, bytedance, fasthttp, api, high-performance
