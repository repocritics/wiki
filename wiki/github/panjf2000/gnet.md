# panjf2000/gnet

> An event-loop networking framework for Go that trades the goroutine-per-connection model for epoll/kqueue reactors, aiming at high connection counts with low memory.

[GitHub repo](https://github.com/panjf2000/gnet) ·
[Official website](https://gnet.host) ·
[License: Apache-2.0](https://github.com/panjf2000/gnet/blob/dev/LICENSE)

## Overview

gnet is a non-blocking, event-driven networking framework written in pure Go, built directly on `epoll` (Linux), `kqueue` (macOS/*BSD), and equivalents[^1]. It descends from Josh Baker's `evio` and reworks it for higher throughput and more features. The pitch is narrow and honest: for a specific class of workloads — many concurrent connections, memory pressure, custom binary protocols — gnet can beat the standard library's `net` package on both throughput and memory footprint. It is explicitly *not* a replacement for `net`; the maintainer states `net` already does its job well for general use[^1].

The defining tradeoff is the programming model. Go's standard `net` gives you a goroutine per connection and blocking reads that the runtime's own netpoller multiplexes for you — simple, idiomatic, and fine for the overwhelming majority of services. gnet inverts this: you register an `EventHandler`, and gnet calls your callbacks (`OnOpen`, `OnTraffic`, `OnClose`, `OnTick`) from a small pool of event-loop goroutines, each pinned to an `epoll`/`kqueue` instance managing thousands of connections. You gain control over buffers and scheduling; you lose the ability to write straight-line blocking code, because anything that blocks a callback blocks every connection on that loop.

gnet operates purely at the transport layer — TCP, UDP, and Unix domain sockets. It ships no HTTP, WebSocket, or RPC layer; you implement application protocols on top of it. As of 2026 it is one of the most-starred Go networking libraries (11k+ stars) and is used in production at several large Chinese tech companies (Tencent, ByteDance, iQIYI, Xiaomi, and others per the README)[^1].

## Getting Started

```bash
go get -u github.com/panjf2000/gnet/v2
```

A minimal echo server. Note the import path ends in `/v2` — v1 and v2 are separate modules with incompatible APIs:

```go
package main

import (
	"log"

	"github.com/panjf2000/gnet/v2"
)

type echoServer struct {
	gnet.BuiltinEventEngine // provides no-op defaults for the EventHandler interface
}

func (es *echoServer) OnTraffic(c gnet.Conn) gnet.Action {
	buf, _ := c.Next(-1) // read everything currently buffered
	c.Write(buf)         // echo it back
	return gnet.None
}

func main() {
	echo := new(echoServer)
	// multicore spins up one event-loop per CPU core
	log.Fatal(gnet.Run(echo, "tcp://:9000", gnet.WithMulticore(true)))
}
```

`gnet.Run` blocks until the engine stops. Callbacks return a `gnet.Action` (`None`, `Close`, `Shutdown`) to steer the connection or the whole server.

## Architecture / How It Works

gnet implements a **multi-reactor** model:

- One (or more) **acceptor loop** accepts new connections on the listener socket.
- Accepted connections are load-balanced onto a pool of **sub event-loops** (`WithMulticore(true)` creates `GOMAXPROCS` of them). Each sub-loop owns a fixed set of connections and runs its own `epoll_wait`/`kevent` in a dedicated goroutine.
- When a socket becomes readable, the owning loop reads into a per-connection inbound buffer and invokes `OnTraffic`. Writes go through an outbound buffer flushed when the socket is writable.

Because a connection lives entirely on one loop goroutine, gnet advertises a **lock-free runtime** — there is no cross-loop synchronization on the hot path[^1]. The cost is that `Conn` is not safe to touch from arbitrary goroutines. The escape hatches are `Conn.AsyncWrite`, `AsyncWritev`, and `Wake`, which are the thread-safe ways to feed data or re-trigger a callback from outside the loop.

Buffer management is a first-class concern. gnet provides an elastic ring buffer, a linked-list buffer, and an elastic-mixed buffer so per-connection memory grows and shrinks with traffic rather than pre-allocating fixed windows[^1]. Read APIs (`Next`, `Peek`, `Read`) let a protocol parser consume bytes without copying more than necessary.

Other built-ins: a goroutine pool via the author's own `ants` library[^2] for offloading blocking work, pluggable load-balancing (`Round-Robin`, `Source-Addr-Hash`, `Least-Connections`), a periodic `OnTick` timer, edge-triggered I/O mode, binding multiple addresses, and a `gnet` client for outbound connections.

## Production Notes

**Never block in a callback.** This is the single most important operational rule and the most common footgun. Callbacks run on the event-loop goroutine; a slow database call or CPU-heavy parse inside `OnTraffic` stalls every other connection on that loop. Offload to the `ants` pool (or your own workers) and return data via `Conn.AsyncWrite`. New users coming from `net` routinely miss this and see mysterious tail-latency spikes under load.

**`Conn` is single-loop-affine.** Do not stash a `Conn` and write to it from another goroutine directly — use `AsyncWrite`/`AsyncWritev`/`Wake`. Ignoring this leads to data races and buffer corruption that surface only under concurrency.

**No TLS.** gnet has no built-in TLS; it sits on the roadmap alongside `io_uring` and KCP but is not shipped[^1]. Layering TLS yourself is genuinely hard here because the event-loop buffer model does not compose cleanly with `crypto/tls`'s blocking `net.Conn` abstraction. If you need TLS at the framework layer, gnet is a poor fit today — evaluate `lesismal/nbio`, which supports non-blocking TLS.

**Windows is dev-only.** The README explicitly states the Windows build is for development and testing, not production[^1]. Real deployments target Linux (`epoll`); *BSD/macOS (`kqueue`) are supported but less exercised at scale.

**You own the protocol.** There is no HTTP or WebSocket layer. The TechEmpower benchmark HTTP handler is described by the maintainer as "half-baked and fine-tuned for benchmark purposes only and far from production-ready"[^1] — do not lift it into a real service. Framing, backpressure, and length-prefixing are your responsibility.

**v1 → v2 was a breaking rewrite.** v2 changed the `EventHandler` interface (the old `React` callback became `OnTraffic`, engine lifecycle changed) and lives at a separate `/v2` module path. Pin the major version and read the migration notes before upgrading; the two are not drop-in compatible.

**Interpreting the benchmarks.** The headline TechEmpower and echo numbers come from tuned, single-purpose setups. gnet's real advantage shows up with large connection counts and small-to-medium packets where goroutine stack overhead and GC pressure hurt the `net` model. For low-connection or request/response HTTP services, standard `net`/`net/http` is usually simpler and fast enough — the gnet win narrows or disappears.

## When to Use / When Not

**Use when:**
- You are handling tens of thousands to millions of concurrent connections and memory-per-connection matters.
- You are building a custom binary protocol at the transport layer (game servers, IM/push, proxies, gateways).
- You need fine control over buffers and connection scheduling that `net` abstracts away.
- Your target is Linux and you can invest in the event-loop mental model.

**Avoid when:**
- You are building a normal HTTP/JSON service — use `net/http` or `fasthttp`; gnet gives you nothing there but complexity.
- You need TLS at the framework layer.
- Your team is unfamiliar with reactor/event-loop programming and blocking-callback hazards.
- You need to run in production on Windows.

## Alternatives

- cloudwego/netpoll — ByteDance's event-loop library underpinning the Kitex RPC framework; use it instead when you are already in the CloudWeGo/Kitex ecosystem.
- lesismal/nbio — non-blocking Go networking with built-in TLS and HTTP; use it instead when you need TLS or an HTTP server on top of the async model.
- tidwall/evio — the ancestor gnet derives from; use it only for a smaller, simpler codebase, as it is less actively developed and slower.
- valyala/fasthttp — use it instead when your actual need is a fast HTTP server, not a raw transport framework.
- golang net (standard library) — use it instead for almost everything else: goroutine-per-connection is simpler, idiomatic, and fast enough for the vast majority of services.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-02 | Repository created; derived from `evio`[^1]. |
| v1.0.0 | 2020 | First stable release; TechEmpower plaintext ranking publicized[^3]. |
| v2.0.0 | 2022 | Major rewrite: new `EventHandler` API (`OnTraffic`), multiple event-loops, elastic buffers, `/v2` module path. |
| v2.x | 2023–2026 | Edge-triggered I/O, multiple-address binding, registering new connections to loops; `io_uring`/TLS/KCP on roadmap[^1]. |

## References

[^1]: gnet README and feature list. https://github.com/panjf2000/gnet
[^2]: `ants` — goroutine pool library by the same author, used internally by gnet. https://github.com/panjf2000/ants
[^3]: TechEmpower Framework Benchmarks, plaintext test. https://www.techempower.com/benchmarks/

## Tags

go, golang, networking, event-loop, epoll, kqueue, non-blocking, reactor, tcp, high-performance, transport-layer
