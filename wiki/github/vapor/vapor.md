# vapor/vapor

> A server-side Swift HTTP web framework built on SwiftNIO — the de facto choice for writing web backends in Swift.

[GitHub repo](https://github.com/vapor/vapor) ·
[Official website](https://vapor.codes) ·
[License: MIT](https://github.com/vapor/vapor/blob/main/LICENSE)

## Overview

Vapor is an HTTP web framework for Swift, first released in 2016[^1]. It sits on top
of Apple's SwiftNIO event-driven networking library and provides routing, middleware,
content negotiation, an async request/response model, and a dependency-injection
container. Alongside Hummingbird it is one of the two frameworks that keep server-side
Swift viable outside Apple's own platforms — and with ~26k stars it is the larger and
older of the two, with the deeper ecosystem (Fluent ORM, Leaf templating, JWT, Queues).

The defining tension of Vapor is the same as server-side Swift itself: you get a
statically typed, ahead-of-time compiled, low-footprint runtime with first-class
async/await, but you pay for it with a comparatively small library ecosystem, long
compile times, and a deployment story that assumes you are comfortable shipping Linux
containers. The framework is mature and stable, but the pool of Swift backend engineers
is far smaller than for Node, Go, or the JVM, and that shapes hiring, third-party
support, and how much you will end up writing yourself.

Vapor 4 — the current major line — targets Swift 6 and has moved fully to
async/await[^2]. Earlier versions exposed SwiftNIO's `EventLoopFuture` directly; treat
the large body of callback-style tutorial material predating that migration as legacy.

## Getting Started

```bash
# Install the Vapor toolbox (macOS, via Homebrew)
brew install vapor
vapor new hello -n          # scaffold a project, no template prompts
cd hello
swift run                   # builds and starts the server on :8080
```

```swift
// Sources/App/routes.swift
import Vapor

func routes(_ app: Application) throws {
    app.get("hello") { req async -> String in
        "Hello, world!"
    }

    app.get("users", ":id") { req async throws -> User in
        let id = try req.parameters.require("id", as: UUID.self)
        return try await User.find(id, on: req.db)
            .unwrap(or: Abort(.notFound))
    }
}
```

Routes are registered against the `Application`; each handler receives a `Request` and
returns anything conforming to `ResponseEncodable` (including `Codable` types, which are
JSON-encoded automatically via the `Content` protocol).

## Architecture / How It Works

Vapor is a layered stack, and understanding the layers explains most of its behavior:

- **SwiftNIO** (`apple/swift-nio`) is the foundation — a non-blocking, event-loop-based
  networking library modeled loosely on Netty. Vapor does not hide NIO entirely; event
  loops, `ByteBuffer`, and `EventLoopFuture` still surface in performance-sensitive or
  lower-level code.
- **Vapor core** adds HTTP routing (a trie-based router), middleware chains, the
  `Content` system for typed request/response bodies, sessions, and an `Application`
  container that doubles as a service/dependency registry via key-path `Storage`.
- **Fluent** (`vapor/fluent`) is the ORM, shipped as a separate package with per-database
  driver packages for PostgreSQL, MySQL, SQLite, and MongoDB[^3]. Models are Swift
  classes annotated with property wrappers (`@ID`, `@Field`, `@Parent`, `@Children`);
  migrations are hand-written Swift types.
- **Leaf** (`vapor/leaf`) is the optional server-side templating engine. Many Vapor apps
  skip it and serve JSON only.

The whole thing is Swift Package Manager only — there is no CocoaPods or Carthage path,
and the `Package.swift` manifest is the single source of dependency truth.

The async model is the most important architectural fact. Vapor 4's earlier releases were
built on `EventLoopFuture<T>` chains; current Vapor bridges those to Swift's native
`async`/`await` and structured concurrency. Under Swift 6's strict concurrency checking,
`Sendable` conformance is enforced at compile time, which surfaces real data-race bugs but
also produces a wave of warnings when porting older code.

## Production Notes

**Deployment means Linux containers.** The realistic production target is a Docker image
built `FROM swift:...` and run on Ubuntu; the toolbox scaffolds a multi-stage Dockerfile
for exactly this. Development is usually on macOS, but the runtime and any platform-
conditional code must be validated on Linux, because Foundation and the Swift standard
library have historically had subtle behavioral differences between the two[^4]. Do not
assume "works on my Mac" implies "works in the container."

**Compile times are a real cost.** Swift's type inference and whole-module optimization
make clean release builds slow, and CI without a warm build cache can spend several
minutes just compiling before tests run. Cache `.build` aggressively and use multi-stage
Docker layering so dependency compilation is not repeated on every code change.

**Concurrency migration is the current upgrade pain.** Moving an existing codebase to
Swift 6 language mode / strict concurrency is the dominant Vapor upgrade task right now.
Expect to add `Sendable` conformances, mark actors, and rework any code that shared
mutable state across event loops. This is genuine work, not a mechanical bump.

**Fluent footguns.** As with any ORM, N+1 queries are easy to introduce via lazy
relationship loading; use `.with(\.$relation)` eager loading deliberately. Migrations are
Swift code that must be registered in order, and there is no built-in "auto-migrate on
boot in production" that you should trust for schema-critical changes — run migrations as
an explicit deploy step (`swift run App migrate`).

**Ecosystem gaps.** Outside the Vapor-maintained packages you will find fewer, less-
maintained third-party libraries than in mainstream backend ecosystems; integrations you
would take for granted elsewhere (vendor SDKs, observability exporters) are often thin
community wrappers or absent. Budget for writing more glue yourself. Windows and other
non-Linux targets remain far behind macOS/Linux — do not plan production on them.

## When to Use / When Not

**Use when:**
- Your team already writes Swift (iOS/macOS) and wants to share models and language across
  client and server.
- You want a compiled, low-memory-footprint backend with modern async/await and are happy
  deploying Linux containers.
- You need a full-stack Swift framework with an ORM, auth, JWT, and queues rather than
  assembling primitives yourself.

**Avoid when:**
- You need a large hiring pool or a deep third-party library ecosystem — Node, Go, or the
  JVM will serve you better.
- Your team has no Swift experience; the language and its concurrency model are a real
  learning investment for a backend-only project.
- You want the leanest possible SwiftNIO framework with minimal abstraction — Hummingbird
  is closer to the metal.
- Fast iterative compile-test loops on a large codebase are critical and you cannot absorb
  Swift's build times.

## Alternatives

- hummingbird-project/hummingbird — newer, lighter server-side Swift framework on the same
  SwiftNIO base; use it when you want fewer abstractions and a smaller dependency surface.
- apple/swift-nio — the layer Vapor is built on; use it directly when you are writing a
  protocol server or proxy rather than a conventional web app.
- Kitura (Kitura/Kitura) — IBM's server-side Swift framework; effectively unmaintained,
  so avoid for new work but relevant when inheriting legacy code.
- expressjs/express — reach for Node/Express instead when hiring pool and ecosystem
  breadth outweigh the benefits of a compiled, typed runtime.
- gin-gonic/gin — a Go alternative when you want a compiled backend with a much larger
  talent pool and simpler deployment story than Swift's.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016 | First release; early pre-SwiftNIO HTTP stack[^1]. |
| 2.0 | 2017 | API overhaul while server-side Swift was still stabilizing. |
| 3.0 | 2018 | Rebuilt on Apple's SwiftNIO; `EventLoopFuture`-based async. |
| 4.0 | 2020 | Current major line; Swift Package Manager-only, revamped API[^2]. |
| 4.x | 2021–2024 | Migration to native async/await, then Swift 6 / strict concurrency support. |

## References

[^1]: Vapor project history and origin (2016). https://vapor.codes
[^2]: Vapor 4 documentation ("Getting Started" / release line). https://docs.vapor.codes/4.0/
[^3]: Fluent ORM and its database drivers. https://docs.vapor.codes/fluent/overview/
[^4]: Swift on the server / Linux deployment guidance. https://docs.vapor.codes/deploy/docker/

## Tags

swift, server-side-swift, web-framework, http, backend, swiftnio, orm, async-await, rest-api, mit-license
