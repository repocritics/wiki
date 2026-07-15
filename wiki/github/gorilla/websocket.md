# gorilla/websocket

> A complete, RFC 6455-conformant WebSocket implementation for Go — the ecosystem's long-standing default, now community-maintained after a near-death archival.

[GitHub repo](https://github.com/gorilla/websocket) ·
[Official website](https://gorilla.github.io) ·
[License: BSD-2-Clause](https://github.com/gorilla/websocket/blob/main/LICENSE)

## Overview

gorilla/websocket is a Go library implementing the WebSocket protocol (RFC 6455) for both client and server roles[^1]. First published in 2013, it became the de facto WebSocket layer for the Go ecosystem — used under the hood by countless chat backends, real-time dashboards, Kubernetes tooling, and reverse proxies. It passes the server-side Autobahn Test Suite, which is the reference conformance battery for WebSocket implementations[^2].

The library's defining characteristic is that it is deliberately low-level and unopinionated. It gives you a `Conn` with framed read/write primitives and leaves connection lifecycle, concurrency discipline, ping/pong keepalive, and reconnection entirely to the caller. This is a strength (it composes with `net/http` cleanly and imposes no runtime) and the source of most production bugs written against it (the concurrency rules are strict and unenforced — see Production Notes).

The project's history includes a governance scare worth knowing: the entire gorilla web toolkit was archived by its maintainer in December 2022, leaving one of Go's most-depended-upon packages formally unmaintained. It was revived under new community maintainers in 2023 and un-archived[^3]. As of this writing the repository is active but low-velocity — commits are infrequent and the API is intentionally frozen, which for a mature protocol library is closer to "done" than to "abandoned." Weigh that against alternatives that receive more frequent updates.

## Getting Started

```bash
go get github.com/gorilla/websocket
```

A minimal echo server built on `net/http`:

```go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	// Default CheckOrigin rejects cross-origin requests. Override
	// deliberately — returning true disables the same-origin guard.
	CheckOrigin: func(r *http.Request) bool { return true },
}

func echo(w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println("upgrade:", err)
		return
	}
	defer conn.Close()

	for {
		mt, msg, err := conn.ReadMessage()
		if err != nil {
			break // client closed or protocol error
		}
		if err := conn.WriteMessage(mt, msg); err != nil {
			break
		}
	}
}

func main() {
	http.HandleFunc("/ws", echo)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Architecture / How It Works

The API surface is small and built around three types:

- **`Upgrader`** — server side. `Upgrader.Upgrade(w, r, responseHeader)` hijacks the underlying TCP connection from the `http.ResponseWriter`, completes the RFC 6455 handshake, and returns a `*Conn`. Because it hijacks, the HTTP handler no longer owns the socket; standard `http` timeouts and middleware after the upgrade do not apply.
- **`Dialer`** — client side. `Dialer.Dial(url, header)` opens a connection and returns a `*Conn` plus the handshake `*http.Response`.
- **`Conn`** — the full-duplex connection. Two API layers: message-level (`ReadMessage`/`WriteMessage`, which buffer a whole message) and streaming (`NextReader`/`NextWriter`, which expose `io.Reader`/`io.WriteCloser` for framing large payloads without holding them in memory).

Control frames (ping, pong, close) are handled through installed callbacks: `SetPingHandler`, `SetPongHandler`, `SetCloseHandler`. Critically, incoming control frames are only processed while a read is in progress — the library does not run a background goroutine. If you want to answer pings, you must keep calling a read method. The default ping handler replies with a pong automatically; the default pong handler is a no-op, so read deadlines for liveness detection are your responsibility via `SetReadDeadline` inside the pong handler.

The concurrency contract is the single most important internal detail: **one concurrent reader and one concurrent writer**. All reads must come from one goroutine (or be externally serialized), all writes from one goroutine. `ReadMessage`, `NextReader`, and `SetReadDeadline` are the "read" family; `WriteMessage`, `NextWriter`, `WriteControl`, and `SetWriteDeadline` are the "write" family. `WriteControl` is the one exception — it is safe to call concurrently with other writes, which is how you send pings from a separate goroutine.

permessage-deflate compression (RFC 7692) is supported but marked experimental in the docs, with a limited negotiation subset; `EnableCompression` on the `Upgrader`/`Dialer` plus `Conn.SetCompressionLevel` control it.

## Production Notes

**The concurrency rule is unenforced and will corrupt frames if violated.** Two goroutines calling `WriteMessage` on the same `Conn` interleave frame bytes on the wire, producing protocol errors the remote side sees as a broken connection. The idiomatic fix is the "single writer goroutine + buffered channel" pattern: all application code sends to a channel, one goroutine drains it and writes. The canonical `examples/chat` in the repo demonstrates exactly this hub/client split — read it before building anything nontrivial.

**Keepalive is manual.** There is no automatic heartbeat. Production deployments must implement a ping/pong loop: a ticker sends `WriteControl(PingMessage, ...)` on an interval, and the read side calls `SetReadDeadline` in the pong handler so a dead peer eventually trips a read timeout. Forgetting this leaves half-open connections that consume memory until the OS TCP keepalive (often 2 hours) reaps them.

**No context.Context support.** The API predates context and uses deadlines (`SetReadDeadline`/`SetWriteDeadline`) rather than `context.Context`. There is no `ReadMessage(ctx)`. If your codebase is context-first, this is friction and a common reason teams migrate to coder/websocket.

**Origin checking defaults to same-origin.** `Upgrader` with a nil `CheckOrigin` rejects requests whose `Origin` header does not match the `Host`. Many tutorials paper over this by returning `true` unconditionally — which disables CSRF protection for the WebSocket endpoint. Set an explicit allowlist in production.

**Buffer sizing and memory.** `ReadBufferSize`/`WriteBufferSize` (default 4096 if zero) are allocated per connection. At high connection counts this dominates memory; use `WriteBufferPool` (a `sync.Pool`) to share write buffers across connections. `SetReadLimit` caps message size to prevent a malicious peer from forcing unbounded allocation.

**One-reader model limits raw connection scale.** Because each connection needs its own reading goroutine, C10k-plus servers pay one goroutine (and its stack) per connection. For hundreds of thousands of mostly-idle connections, epoll-based libraries (gobwas/ws, lesismal/nbio) that decouple reads from goroutines scale further.

## When to Use / When Not

**Use when:**
- You want the well-worn, battle-tested default and are already on `net/http`.
- You need proven RFC 6455 conformance (Autobahn-passing) without evaluating options.
- Connection counts are moderate (thousands to low tens of thousands) and a goroutine-per-connection model is acceptable.
- You value API stability over active feature development.

**Avoid when:**
- You want a context.Context-native API and a smaller surface — coder/websocket fits better.
- You are building a massive-fanout server (100k+ concurrent sockets) where per-connection goroutines are too costly.
- You need first-class, non-experimental compression or HTTP/2-era transport features.
- You want a maintainer actively shipping — velocity here is low by design.

## Alternatives

- coder/websocket (formerly nhooyr.io/websocket) — minimal, context.Context-first, `net/http`-idiomatic. Use when you want a modern API and are starting fresh.
- gobwas/ws — low-level, zero-allocation framing with optional epoll. Use when you need maximum throughput and are willing to build the connection layer yourself.
- lesismal/nbio — nonblocking, epoll/kqueue-based networking. Use when you must hold hundreds of thousands of concurrent connections without a goroutine each.
- golang.org/x/net/websocket — the old std-adjacent package; effectively deprecated. Do not use for new code; its own docs point users to alternatives.
- fasthttp/websocket — a gorilla fork adapted to fasthttp. Use when your stack is already built on fasthttp instead of net/http.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-10 | First public release of the package[^1]. |
| v1.4.2 | 2020-03 | Last release before the maintenance gap. |
| (archived) | 2022-12 | gorilla toolkit archived; package formally unmaintained[^3]. |
| (revived) | 2023 | Un-archived under new community maintainers[^3]. |
| v1.5.x | 2023–2024 | Post-revival releases; API kept stable, minor fixes. |

## References

[^1]: Package documentation and history, pkg.go.dev. https://pkg.go.dev/github.com/gorilla/websocket
[^2]: Autobahn WebSocket Test Suite (conformance battery the package passes on the server side). https://github.com/crossbario/autobahn-testsuite
[^3]: Gorilla Web Toolkit revival announcement (the toolkit was archived in December 2022 and un-archived in 2023). https://github.com/gorilla#gorilla-toolkit
[^4]: Repository README and status notice — API described as stable. https://github.com/gorilla/websocket

## Tags

go, golang, websocket, rfc6455, networking, real-time, http, client-server, low-level, protocol
