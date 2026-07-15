# grpc/grpc-swift

> The gRPC implementation for Swift, built on SwiftNIO — now in maintenance mode as its v2 rewrite takes over.

[GitHub repo](https://github.com/grpc/grpc-swift) ·
[gRPC project](https://grpc.io) ·
[License: Apache-2.0](https://github.com/grpc/grpc-swift/blob/main/LICENSE)

## Overview

grpc-swift is the Swift implementation of gRPC: a code generator (a `protoc`
plugin) plus runtime libraries for building gRPC clients and servers[^1]. This
repository holds **gRPC Swift 1**, which the maintainers have explicitly moved
into maintenance mode — only bug fixes and security fixes land here, and the
window of supported Swift toolchains shrinks with each new Swift release[^2].
New work happens in a separate repository, grpc/grpc-swift-2, which is a
ground-up rewrite rather than a version bump[^3].

The defining fact about the 1.x line is that it was itself a rewrite. Early
grpc-swift (pre-1.0) wrapped the C-based gRPC core library, inheriting its build
complexity and threading model. The 1.0 line dropped that dependency and
reimplemented the stack in pure Swift on top of SwiftNIO, Apple's event-loop
networking library[^1]. That gave it a Linux-and-Apple story with no C
toolchain, at the cost of tying every application to NIO's `EventLoopFuture`
concurrency model — which later Swift structured concurrency made feel dated,
and which is the main reason a v2 rewrite exists at all.

The audience today is narrow but real: teams with existing gRPC Swift 1
deployments, and code already built on SwiftNIO `EventLoopFuture` that is not
ready to migrate. Everyone starting fresh is directed to v2.

## Getting Started

Add the package and the plugins via Swift Package Manager:

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/grpc/grpc-swift.git", from: "1.23.0"),
    .package(url: "https://github.com/apple/swift-protobuf.git", from: "1.28.0"),
],
```

Generate stubs from a `.proto` with `protoc` plus both plugins (messages come
from SwiftProtobuf, service stubs from grpc-swift):

```bash
protoc echo.proto \
  --swift_out=. --swift_opt=Visibility=Public \
  --grpc-swift_out=. --grpc-swift_opt=Visibility=Public,Client=true,Server=true
```

A minimal async client call:

```swift
import GRPC
import NIOPosix

let group = MultiThreadedEventLoopGroup(numberOfThreads: 1)
defer { try? group.syncShutdown() }

let channel = try GRPCChannelPool.with(
    target: .host("localhost", port: 1234),
    transportSecurity: .plaintext,
    eventLoopGroup: group
)
defer { try? channel.close().wait() }

let echo = Echo_EchoAsyncClient(channel: channel)
let reply = try await echo.get(.with { $0.text = "hello" })
print(reply.text)
```

## Architecture / How It Works

The runtime sits on the SwiftNIO stack: HTTP/2 framing via `nio-http2`, TLS via
`nio-ssl`, and everything scheduled on a `MultiThreadedEventLoopGroup`. A gRPC
call is an HTTP/2 stream; the four call shapes (unary, client-streaming,
server-streaming, bidirectional) map onto stream lifecycles with
length-prefixed protobuf frames.

Code generation is two-stage and this trips people up. `protoc-gen-swift`
(from apple/swift-protobuf) emits the message types; `protoc-gen-grpc-swift`
(this repo) emits the service client and server stubs that reference those
messages[^1]. Both must run, and their `Visibility`/`Access` options must
agree. A SwiftPM build-tool plugin exists to run generation at build time
instead of committing generated files.

There are two overlapping API surfaces:

- **`EventLoopFuture`-based** — the original API. Calls return futures resolved
  on an event loop; you compose with `.flatMap`, `.map`, and friends, and you
  must never block the event-loop thread.
- **`async`/`await`-based** — the `GRPCAsync*` types (async clients, async
  server handlers, `RPCAsyncSequence` streams) added later in the 1.x line[^4].
  These bridge onto the same NIO core, so the event-loop group is still present
  underneath even though your call sites look like structured concurrency.

Cross-cutting concerns go through **interceptor chains**: `ClientInterceptor`
and `ServerInterceptor` subclasses see each part of a call (metadata, message,
end) and can add auth, logging, retries, or tracing. `GRPCChannelPool` manages a
pool of HTTP/2 connections with configurable connection backoff and keepalive.

v2 (grpc/grpc-swift-2) reorganizes all of this into separate packages —
transport-agnostic core, a NIO HTTP/2 transport, protobuf serialization,
extras — and is structured-concurrency-first. Module names differ (`GRPCCore`,
`GRPCNIOTransportHTTP2`, `GRPCProtobuf`), so it is not source-compatible with
`import GRPC`[^3].

## Production Notes

**Maintenance mode is the headline caveat.** No new features will land here, and
the supported-Swift matrix in the README contracts as Swift releases advance —
a toolchain that works today may be dropped in a later 1.x support window[^2].
Treat 1.x as a stable target to keep running, not to build new systems on.

**The v1 → v2 move is a migration, not an upgrade.** Different repository,
different modules, different concurrency model. There is no drop-in path; you
rewrite call sites and regenerate stubs with the v2 plugin. Budget it as
project work, not a dependency bump.

**Do not block the event loop.** With the `EventLoopFuture` API this is the
classic footgun: any synchronous, blocking, or CPU-heavy work inside a handler
or a future callback stalls every connection sharing that loop. Offload to a
`DispatchQueue` or `NIOThreadPool`, or use the async API and keep blocking work
off the cooperative thread pool.

**Codegen version skew.** Because message and stub generation are two separate
plugins, a mismatch between the installed `protoc-gen-grpc-swift`, the
`swift-protobuf` version, and the runtime library produces confusing compile
errors. Pin all three and regenerate together.

**Platforms.** Apple platforms and Linux are supported through SwiftNIO; there
is no Windows story to rely on. TLS depends on `nio-ssl` (BoringSSL), which adds
build weight and can complicate static linking and container images. Reflection,
health, and similar add-ons live outside the core library and must be wired in
explicitly.

## When to Use / When Not

**Use when:**
- You maintain an existing gRPC Swift 1 service and want stability, not new APIs.
- Your codebase is already SwiftNIO / `EventLoopFuture`-native and integrates at
  the event-loop level.
- You need a proven, plain-Swift gRPC stack with no C toolchain dependency.

**Avoid when:**
- You are starting a new project — use grpc/grpc-swift-2 instead.
- You want a structured-concurrency-first design without an event-loop group
  underneath.
- You need a long, predictable support window across future Swift releases.
- You only need a client and want the lightest possible dependency (consider
  Connect-Swift).

## Alternatives

- grpc/grpc-swift-2 — the official successor; use for any new gRPC Swift work.
- connectrpc/connect-swift — use when you want a client-only library speaking
  gRPC, gRPC-Web, or the Connect protocol over URLSession, with no server side.
- apple/swift-protobuf — use when you need Protocol Buffers messages but not RPC.
- grpc/grpc (Core + Objective-C) — use for legacy iOS apps already on the
  CocoaPods `gRPC-Core`/`gRPC-ProtoRPC` stack.
- vapor/vapor or a plain REST/JSON API — use when you do not actually need gRPC
  and HTTP/1 + JSON would be simpler to operate.

## History

| Version | Date | Notes |
|---------|------|-------|
| pre-1.0 | 2016–2019 | Wrapped the C-based gRPC core library[^1]. |
| 1.0 | 2021 | Pure-Swift rewrite on SwiftNIO; dropped the C dependency[^1]. |
| 1.8 | 2022 | `async`/`await` API surface (`GRPCAsync*`) added[^4]. |
| 1.x | ongoing | Maintenance mode — bug and security fixes only[^2]. |
| v2 | 2025 | Separate grpc-swift-2 repo; modular, concurrency-first rewrite[^3]. |

## References

[^1]: gRPC Swift README and package documentation. https://github.com/grpc/grpc-swift
[^2]: gRPC Swift README, "Versions" — 1.x is in maintenance mode with a shrinking Swift support window. https://github.com/grpc/grpc-swift#versions
[^3]: gRPC Swift 2 repository. https://github.com/grpc/grpc-swift-2
[^4]: SwiftNIO and gRPC Swift async/await support. https://swiftpackageindex.com/grpc/grpc-swift/main/documentation/grpccore

## Tags

swift, grpc, rpc, protocol-buffers, swiftnio, http2, code-generation, networking, apple, maintenance-mode
