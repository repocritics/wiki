# apple/swift-nio

> A low-level, event-driven, non-blocking network application framework for Swift — "Netty, but for Swift."

[GitHub repo](https://github.com/apple/swift-nio) ·
[Documentation](https://swiftpackageindex.com/apple/swift-nio/documentation) ·
[License: Apache-2.0](https://github.com/apple/swift-nio/blob/main/LICENSE.txt)

## Overview

SwiftNIO is Apple's networking foundation for the server-side Swift ecosystem, first
released in 2018[^1]. It is deliberately low-level: it provides event loops, channels,
byte buffers, and a handler pipeline, and explicitly does *not* provide web frameworks,
routing, or high-level protocol clients. Its self-description — "like Netty, but written
for Swift" — is accurate; the reactor pattern, the channel/handler pipeline, and much of
the vocabulary come directly from Java's Netty[^2]. If you write Swift on the server, you
are almost certainly using SwiftNIO, usually transitively through Vapor, Hummingbird, gRPC
Swift, or AsyncHTTPClient rather than directly.

The framework targets the case where a thread-per-connection model breaks down: many
connections, each with low individual utilization. Instead of blocking a thread per socket,
SwiftNIO runs a small number of event loops (roughly one per core), each driving an
`epoll` (Linux) or `kqueue` (Darwin/BSD) selector, and dispatches readiness events into
user-supplied handlers. It is a graduated project in the Swift Server Workgroup (SSWG)
incubation process[^3] — the closest thing server-Swift has to a stability guarantee.

The defining tension in 2026 is SwiftNIO's *two* concurrency models. NIO predates Swift's
native `async`/`await`, so its core is built on callbacks and its own `EventLoopFuture`
type. Since Swift 5.5, NIO has added bridges (`NIOAsyncChannel`, async sequence wrappers)
so that new code can consume it with structured concurrency — but the futures-based core
remains, and mixing the two models is the most common source of confusion and subtle bugs.

## Getting Started

Add it as a Swift Package Manager dependency, then depend on the `NIOCore` and
`NIOPosix` products:

```swift
// Package.swift
.package(url: "https://github.com/apple/swift-nio.git", from: "2.0.0"),
```

A minimal TCP echo server using the classic pipeline API:

```swift
import NIOCore
import NIOPosix

final class EchoHandler: ChannelInboundHandler {
    typealias InboundIn = ByteBuffer
    typealias OutboundOut = ByteBuffer

    func channelRead(context: ChannelHandlerContext, data: NIOAny) {
        context.write(data, promise: nil)   // echo the bytes straight back
    }
    func channelReadComplete(context: ChannelHandlerContext) {
        context.flush()
    }
}

let group = MultiThreadedEventLoopGroup(numberOfThreads: System.coreCount)
defer { try! group.syncShutdownGracefully() }

let channel = try ServerBootstrap(group: group)
    .serverChannelOption(ChannelOptions.backlog, value: 256)
    .childChannelInitializer { channel in
        channel.pipeline.addHandler(EchoHandler())
    }
    .bind(host: "127.0.0.1", port: 8080)
    .wait()

print("Listening on \(channel.localAddress!)")
try channel.closeFuture.wait()
```

## Architecture / How It Works

SwiftNIO applications are assembled from eight core types, all provided by `NIOCore`:
`EventLoopGroup`, `EventLoop`, `Channel`, `ChannelHandler`, `Bootstrap`, `ByteBuffer`,
`EventLoopFuture`, and `EventLoopPromise`.

**Event loops.** The `EventLoop` is a thread spinning in a dispatch loop, waking on socket
readiness. `MultiThreadedEventLoopGroup` spawns POSIX threads and pins one selector-backed
loop to each. The critical invariant: nearly every operation on a `Channel` runs *on its
event loop's thread*. This is how SwiftNIO gets thread-safety without locks — but it means
one blocking call inside a handler stalls every connection sharing that loop.

**Channels and the pipeline.** Each socket is owned by a `Channel`, which carries a
`ChannelPipeline`: an ordered chain of `ChannelHandler`s. Inbound events (reads, close)
flow front-to-back; outbound events (writes, connects) flow back-to-front. Handlers are
small, single-purpose, and composable — an HTTP decoder, a TLS handler, a framer — each
transforming events and forwarding via a `ChannelHandlerContext`, exactly as in Netty.
`ByteBuffer`, a copy-on-write reference-counted byte container with separate reader/writer
indices, is the payload type throughout, tuned to avoid hot-path allocations.

**Modularization.** The package splits abstractions from implementation: `NIOCore` holds
protocol-level abstractions with no I/O, `NIOPosix` holds the real `epoll`/`kqueue` loops
and socket channels, and `NIOEmbedded` provides `EmbeddedChannel`/`EmbeddedEventLoop` for
deterministic tests with no sockets[^4]. Extension libraries should depend only on
`NIOCore`. Sibling repos add TLS (`swift-nio-ssl`), HTTP/2 (`swift-nio-http2`), SSH
(`swift-nio-ssh`), Apple's Network.framework transport (`swift-nio-transport-services`),
and in-development QUIC/HTTP3.

**Futures vs. async/await.** `EventLoopFuture`/`EventLoopPromise` are NIO's original
asynchrony model, each tied to a specific event loop. `NIOAsyncChannel` and async-sequence
wrappers bridge streams into Swift structured concurrency, but the bridge is leaky:
back-pressure, cancellation, and executor identity all behave differently across it.

## Production Notes

**Never block an event loop.** This is the number-one operational footgun. Any synchronous
disk I/O, `Data(contentsOf:)`, DNS lookup, database call, or `.wait()` inside a handler
blocks the entire event loop and every connection on it. Offload to `NIOThreadPool` or a
background executor. `.wait()` is for top-level `main` glue only and will *deadlock* if
called on an event loop thread.

**You will rarely use it directly.** Most teams should reach for Vapor or Hummingbird, not
raw NIO. Writing correct handlers requires understanding the execution model, reference
semantics of `ByteBuffer`, and pipeline ordering. Direct NIO use is justified for custom
protocol implementations, proxies, and performance-critical middleware — not application
code.

**Concurrency-model friction.** `NIOAsyncChannel` improved the futures↔`async`/`await`
bridge substantially, but back-pressure propagation across it and cancellation of in-flight
async work both require care; code that `await`s inside a handler is not automatically on
the channel's event loop.

**Swift version churn.** SwiftNIO tracks Swift aggressively — latest plus the two prior
minor releases. NIO 2.87+ requires Swift 6.0 and NIO 2.98+ requires Swift 6.1[^5]. Full
Swift 6 strict-concurrency (`Sendable`) checking surfaced many latent data-race warnings in
downstream code during the 6.0 migration.

**Platform coverage.** Developed and tested on macOS and Linux. On Apple platforms,
`swift-nio-transport-services` swaps the POSIX loop for Network.framework — required for
iOS/tvOS/watchOS. OpenBSD support is experimental (no `_NIOFileSystem`); there is no
official Windows support in the core I/O layer. NIO 1 is end-of-life (no fixes after May
2022) — all new work should target NIO 2, which follows SemVer 2.0.0 against a documented
public-API contract, with a migration guide for the significant API changes[^6].

## When to Use / When Not

**Use when:**
- You are implementing a network protocol (custom binary protocol, proxy, message broker)
  and need direct control over framing, back-pressure, and the socket lifecycle.
- You are building the networking layer *of* a framework or client library.
- You need predictable, high-connection-count performance without thread-per-connection cost.
- You want Apple-backed, SSWG-graduated infrastructure with a stable public-API guarantee.

**Avoid when:**
- You are building an application (API server, web app) — use Vapor or Hummingbird, which
  wrap NIO with routing, middleware, and higher-level ergonomics.
- Your team is new to non-blocking I/O and event-loop execution models; the learning curve
  is steep and the failure modes (blocked loops, deadlocked `.wait()`) are unforgiving.
- You want a pure `async`/`await` API with no futures — the bridges help, but the core
  concept surface is still futures-first.
- You need Windows as a first-class server target.

## Alternatives

- vapor/vapor — full web framework built on SwiftNIO; use instead when you want routing,
  middleware, ORM, and application ergonomics rather than raw sockets.
- hummingbird-project/hummingbird — lighter, modular server framework on NIO; use when you
  want a smaller, async/await-native surface than Vapor.
- netty/netty — the JVM framework SwiftNIO is modeled on; use when your stack is Java/Kotlin.
- tokio-rs/tokio — the equivalent async runtime and networking foundation for Rust.
- swift-server/async-http-client — if you only need an HTTP client, use this rather than
  building one on NIO yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-03 | Initial public release; reactor/pipeline model[^1]. |
| 2.0 | 2019-03 | Swift 5 baseline; major API revision, current major line. |
| 2.x (modularization) | 2021 | Split into `NIOCore` / `NIOPosix` / `NIOEmbedded`[^4]. |
| 2.x (concurrency) | 2022–2023 | `NIOAsyncChannel` and async-sequence bridges for Swift Concurrency. |
| 2.87.0 | 2025 | Minimum Swift raised to 6.0[^5]. |
| 2.98.0 | 2026 | Minimum Swift raised to 6.1[^5]. |

## References

[^1]: SwiftNIO README, "SwiftNIO is a cross-platform asynchronous event-driven network application framework." https://github.com/apple/swift-nio
[^2]: Netty project — the JVM framework SwiftNIO's pipeline model is derived from. https://netty.io
[^3]: Swift Server Workgroup incubation process (SwiftNIO listed at graduated level). https://github.com/swift-server/sswg/blob/main/process/incubation.md
[^4]: SwiftNIO README, "Repository organization" and module breakdown (`NIOCore`, `NIOPosix`, `NIOEmbedded`). https://github.com/apple/swift-nio#repository-organization
[^5]: SwiftNIO README, "Swift Versions" support table (2.87.0 → Swift 6.0, 2.98.0 → Swift 6.1). https://github.com/apple/swift-nio#swift-versions
[^6]: SwiftNIO NIO 1 → NIO 2 migration guide. https://github.com/apple/swift-nio/blob/main/docs/migration-guide-NIO1-to-NIO2.md

## Tags

swift, networking, event-driven, non-blocking-io, event-loop, server-side-swift, async, framework, apple, high-performance, protocol-implementation
