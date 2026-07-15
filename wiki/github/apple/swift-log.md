# apple/swift-log

> A logging API — not a logging backend — for the Swift server ecosystem, designed so libraries can emit logs without dictating where they go.

[GitHub repo](https://github.com/apple/swift-log) ·
[Official website](https://swiftpackageindex.com/apple/swift-log) ·
[License: Apache-2.0](https://github.com/apple/swift-log/blob/main/LICENSE.txt)

## Overview

SwiftLog is an API package produced under the Swift Server Workgroup (SSWG)[^1]. Its central design decision is separation of concerns: the package defines *how* code emits log messages (the `Logger` value type and the `LogHandler` protocol) but ships only a minimal default implementation of *where* those messages go. A library author depends on the `Logging` product and calls `logger.info(...)`; the application that assembles those libraries decides at startup whether messages are written to stderr, JSON, syslog, a cloud aggregator, or discarded. This is the same "facade" pattern that SLF4J established in the Java ecosystem, and SwiftLog is explicit about that lineage.

The defining tension is that SwiftLog is deliberately small. It is not a batteries-included logging framework: there is no built-in file rotation, no async batching pipeline, no network shipping, no log formatting DSL. Those live in third-party backends. The payoff is that a Swift library can adopt logging without forcing a heavyweight dependency or a formatting opinion onto its consumers; the cost is that "just add logging" for an application means also choosing and wiring a backend, and the ecosystem of backends varies in quality and maintenance.

It is broadly adopted across server-side Swift — SwiftNIO, Vapor, Hummingbird, the AWS Lambda Swift runtime, and gRPC-Swift all emit through it — which makes it a de facto standard for that domain rather than a competitor within it.

## Getting Started

Add the package and depend on the `Logging` product:

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/apple/swift-log", from: "1.6.0")
],
targets: [
    .target(name: "YourApp", dependencies: [
        .product(name: "Logging", package: "swift-log")
    ])
]
```

```swift
import Logging

// Optional: configure the backend ONCE, before any Logger is created.
LoggingSystem.bootstrap(StreamLogHandler.standardError)

let logger = Logger(label: "com.example.YourApp")
logger.info("Application started")
logger.error("Request failed", metadata: ["request-id": "\(UUID())"])

// Loggers are value types; a copy can carry request-scoped metadata.
var reqLogger = logger
reqLogger[metadataKey: "trace-id"] = "abc123"
reqLogger.debug("handling request")   // trace-id attached to this line
```

## Architecture / How It Works

`Logger` is a `struct` — a value type — that wraps a single `LogHandler`. Because it is a value, copying a logger and mutating its metadata (`logger[metadataKey:]`) gives you an independent, request-scoped logger without touching global state. This is the mechanism used to attach per-request or per-connection context as work fans out.

`LogHandler` is the extension point: a protocol with a `log(level:message:metadata:...)` method plus stored `logLevel` and `metadata`. Anyone can implement it; a "backend" is just a `LogHandler` conformance plus a factory.

`LoggingSystem.bootstrap(_:)` installs the global factory that `Logger(label:)` uses to construct handlers. The contract is that it must be called exactly once, at process startup, before any `Logger` is instantiated. It is guarded to prevent multiple bootstraps — a second call traps rather than silently reconfiguring. The default when you never bootstrap is `StreamLogHandler` writing to stderr.

Log levels are the standard seven: `trace`, `debug`, `info`, `notice`, `warning`, `error`, `critical`. Two details matter for performance. First, the message and metadata parameters are `@autoclosure` — string interpolation is only evaluated if the handler will actually emit the line, so a `logger.trace("\(expensiveDescription)")` below threshold costs nothly. Second, level filtering is the handler's responsibility, per logger instance, so different subsystems can run at different verbosity.

Metadata is a recursive value type (`Logger.MetadataValue`: string, array, or dictionary). Version 1.5.0 added `MetadataProvider`[^2], a task-local hook that lets a handler pull ambient context (for example a trace ID from swift-distributed-tracing) at log time without the call site passing it explicitly.

## Production Notes

**Bootstrap ordering is a real footgun.** Because `bootstrap` must run before the first `Logger` is created and traps on a second call, test suites and plugin architectures that construct loggers eagerly can either miss the bootstrap window or crash on a re-bootstrap. Common fixes: bootstrap in `main()` before anything else, and in tests use a bootstrap-once helper or `LoggingSystem.bootstrapInternal` patterns rather than calling the public API repeatedly.

**The label is not a category.** `Logger(label:)` takes a free-form string; it is not a hierarchical subsystem/category the way Apple's `os.Logger` is, and there is no built-in level configuration by label prefix. If you want "set `com.example.db` to debug but everything else to info," you build that routing yourself in a custom handler.

**Synchronous by default.** The core `log` call is synchronous. `StreamLogHandler` writes to stderr under a lock, which serializes output and can become a contention point in very high-throughput services. Production deployments that log heavily typically adopt a backend that buffers and flushes off the hot path rather than relying on the bundled stream handler.

**You must choose a backend deliberately.** For anything beyond stderr — structured JSON, file output with rotation, syslog, or shipping to Datadog/OpenTelemetry/CloudWatch — you depend on a community package. Backend quality and maintenance vary; check recent commits and Swift-version support on the Swift Package Index before adopting one.

**Stable API surface.** SwiftLog has stayed on the 1.x line for its whole life with a strong source-stability commitment, so upgrades within 1.x are low-risk. The practical churn is on the Swift-tools and concurrency side: newer releases raise the minimum Swift version and tighten `Sendable`/strict-concurrency conformance, which can surface warnings when you move to Swift 6 language mode.

## When to Use / When Not

**Use when:**
- You are writing a Swift library and want to emit logs without imposing a backend on consumers.
- You are building a server-side Swift application and want the ecosystem-standard logging seam that SwiftNIO, Vapor, and Hummingbird already speak.
- You want per-request metadata propagation via cheap value-type logger copies.

**Avoid / reconsider when:**
- You are shipping an Apple-platform app (iOS/macOS UI) with no server component — Apple's native `os.Logger` (OSLog) integrates with Console.app, signposts, and the unified logging system that SwiftLog does not.
- You want a turnkey framework with file rotation, formatting, and network shipping out of the box — SwiftLog gives you none of that without a backend.
- You need guaranteed non-blocking, high-volume logging and are not prepared to select/tune a buffered backend.

## Alternatives

- apple/swift-metrics — sibling SSWG package; use it instead when you need counters/gauges/timers rather than textual log events (they are complementary, not substitutes).
- apple/swift-distributed-tracing — use when you need span/trace propagation; pairs with swift-log via MetadataProvider rather than replacing it.
- SwiftyBeaver/SwiftyBeaver — use when you want a self-contained logging framework with built-in destinations (file, cloud) and less wiring, and don't need the API/backend split.
- CocoaLumberjack/CocoaLumberjack — use for mature, batteries-included logging on Apple/Objective-C-heavy codebases.
- Apple `os.Logger` (OSLog, part of the OS) — use for Apple-platform apps that want native Console/Instruments integration instead of a portable server-oriented API.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2019 | Initial release under the Swift Server Workgroup; `Logger` + `LogHandler` API stabilized.[^1] |
| 1.4.x | 2021 | Concurrency/`Sendable` hardening; expanded metadata value support. |
| 1.5.0 | 2022 | `MetadataProvider` — task-local ambient metadata hook.[^2] |
| 1.6.0 | 2024 | Swift 6 / strict-concurrency alignment; current 1.x line.[^3] |

## References

[^1]: Swift Server Workgroup / Swift blog announcement of a standardized logging API. https://www.swift.org/blog/logging/
[^2]: SwiftLog 1.5.0 release notes — MetadataProvider. https://github.com/apple/swift-log/releases
[^3]: apple/swift-log README, Quick Start (`from: "1.6.0"`). https://github.com/apple/swift-log
[^4]: SwiftLog API documentation, Swift Package Index. https://swiftpackageindex.com/apple/swift-log/documentation

## Tags

swift, logging, server-side-swift, logging-api, observability, sswg, apple, structured-logging, facade-pattern, library
