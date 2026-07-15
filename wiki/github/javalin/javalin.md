# javalin/javalin

> A lightweight HTTP and WebSocket layer over embedded Jetty for Java and Kotlin — a library, not a framework, with no annotations, reflection, or DI container.

[GitHub repo](https://github.com/javalin/javalin) ·
[Official website](https://javalin.io) ·
[License: Apache-2.0](https://github.com/javalin/javalin/blob/master/LICENSE)

## Overview

Javalin is a thin routing and request-handling layer that sits on top of embedded Jetty. It began around 2017 as a fork of Per Wendel's Spark (SparkJava), created by David Åse, and diverged into its own codebase optimized for symmetric Java/Kotlin ergonomics[^1]. Its stated design stance is explicit: no annotations, no reflection, no "magic" — you build a server object in code, register lambda handlers, and start it. This makes control flow easy to follow and step-debug, at the cost of the compile-time route validation and convention-driven wiring that annotation frameworks provide.

The audience is teams that want a Spring Boot alternative for small-to-medium HTTP services and want to keep the whole request path readable. It is popular for REST APIs, internal microservices, and WebSocket/SSE endpoints where the Spring ecosystem feels heavy. As of 2026 it has roughly 8.3k stars and 640 forks[^2], a mid-size but stable following — actively maintained (last pushed mid-2026) by a small core team plus a Discord community, not a corporate-backed project.

The defining tension is coupling to Jetty. Javalin does not abstract the server away; a Javalin major release typically tracks a Jetty major, and that transitively decides your servlet namespace and minimum JDK. That is what makes Javalin small and predictable, and also what makes its major upgrades more disruptive than the "just bump the version" experience its API surface suggests.

## Getting Started

Maven:

```xml
<dependency>
    <groupId>io.javalin</groupId>
    <artifactId>javalin</artifactId>
    <version>7.2.2</version>
</dependency>
```

Gradle (Kotlin DSL):

```kotlin
implementation("io.javalin:javalin:7.2.2")
```

Minimal server (Kotlin, current 7.x config-routing API):

```kotlin
import io.javalin.Javalin

fun main() {
    val app = Javalin.create { config ->
        config.routes.get("/") { ctx -> ctx.result("Hello World") }
        config.routes.get("/users/{id}") { ctx ->
            ctx.json(mapOf("id" to ctx.pathParam("id")))
        }
    }.start(7070)
}
```

Note the API moved: earlier majors registered routes on the app instance (`app.get(...)`); 7.x registers them inside the `config.routes` block[^3]. Java usage is identical modulo lambda syntax.

## Architecture / How It Works

Under the hood a Javalin instance owns an embedded Jetty `Server` and a `ServletHandler`. Every request is dispatched to Jetty's thread pool (a `QueuedThreadPool` by default), so the default execution model is thread-per-request and **blocking** — your handler holds a Jetty worker thread for its full duration. Async is opt-in: return a future via `ctx.future { ... }`, or switch the pool to virtual threads with `config.concurrency.useVirtualThreads = true`, which is the intended Loom-era path for I/O-bound handlers.

Routing is a matcher over registered path patterns (`/{param}`, `/<wildcard>`), evaluated per request — there is no code generation or annotation scanning, which is why startup is fast and there is nothing to "warm up." Handlers are plain lambdas receiving a `Context` (`ctx`), a mutable wrapper over the servlet request/response exposing path/query params, body parsing, headers, and the response.

JSON is pluggable through a `JsonMapper` interface; the default implementation is Jackson. In Kotlin you must add `jackson-module-kotlin` for data classes to deserialize, and register `JavaTimeModule` if you serialize `java.time` types — neither is wired automatically. WebSockets, Server-Sent Events, and HTTP/2 are first-class and configured through the same builder. `before`/`after` filters, typed `exception(...)` mappers, and status-code `error(...)` handlers compose the middleware story; there is no separate filter-chain abstraction.

The whole surface is a facade over Jetty rather than an abstraction of it. You can reach the raw Jetty `Server` to configure connectors, SSL (via the SSL plugin), or thread pools directly. This is the source of both Javalin's smallness and its version coupling.

## Production Notes

- **Blocking by default.** A slow downstream call ties up a Jetty worker for its whole duration. Under load this manifests as pool exhaustion and queuing, not CPU saturation. Size the Jetty pool for your concurrency, or move I/O-bound work onto virtual threads / `ctx.future`.
- **Jetty coupling drives upgrades.** Javalin 5 moved to Jetty 11 and the `jakarta.servlet` namespace (from `javax.servlet`), which is a breaking classpath change for anything else depending on old Jetty/servlet. If another dependency pulls a different Jetty major, you get `NoSuchMethodError`/`NoClassDefFoundError` at runtime, not a build error. Pin and dependency-tree-audit Jetty on every Javalin major bump.
- **Config restructures every major.** The configuration API is reorganized across majors (routing moved onto `config.routes` in 7.x; the pre-6 `AccessManager` was replaced by route-role / `beforeMatched` patterns). Each major ships a migration guide because these are not source-compatible. Budget real time for major upgrades even though the runtime is small[^3].
- **JSON gotchas.** Missing `jackson-module-kotlin` produces confusing "cannot construct instance" errors on Kotlin data classes; missing `JavaTimeModule` serializes timestamps as arrays. Both are easy to hit and not surfaced by Javalin itself.
- **No built-in auth/security.** There is no authentication, authorization framework, or CSRF layer. You implement access control in `before` filters or route roles. This is deliberate but means security is entirely your responsibility.
- **Observability is BYO.** Metrics, tracing, and structured request logging are add-ons (Micrometer plugin, request loggers) rather than defaults.

## When to Use / When Not

**Use when:**
- You want a readable, debuggable HTTP/WebSocket service in Java or Kotlin without a DI container or annotation processing.
- You're building small-to-medium REST APIs or microservices and Spring Boot feels oversized.
- You value fast startup and low memory, and want direct access to Jetty when needed.
- You need Java/Kotlin interop parity in the same codebase.

**Avoid when:**
- You want batteries-included: security, data access, config, and a large integration ecosystem out of the box (use Spring Boot).
- You want a fully non-blocking, coroutine- or reactive-native stack (Ktor, http4k, Vert.x fit better).
- You need GraalVM native-image / annotation-driven, AOT-optimized services (Quarkus, Micronaut).
- You cannot tolerate meaningful migration work on major version bumps.

## Alternatives

- spring-projects/spring-boot — use instead when you need the full JVM ecosystem (security, data, actuator, DI) and can accept the weight and startup cost.
- ktorio/ktor — use instead when you want a Kotlin-first, coroutine-native async framework from JetBrains rather than a Jetty facade.
- http4k/http4k — use instead when you want purely functional "server as a function" Kotlin HTTP with pluggable backends and no Jetty lock-in.
- quarkusio/quarkus — use instead when you target GraalVM native images and fast cold starts for serverless/Kubernetes.
- perwendel/spark — Javalin's ancestor; use only for legacy maintenance, as it is largely inactive.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017 | Initial release; diverged from a Spark fork into its own codebase[^1]. |
| 2.0 | 2018 | Internal rewrite; API cleanup. |
| 3.0 | 2019 | Major rewrite of routing and validation. |
| 4.0 | 2021-11 | Config and handler refinements. |
| 5.0 | 2022-09 | Jetty 11 + `jakarta.servlet` namespace; JDK baseline raised[^3]. |
| 6.0 | 2024-01 | Config restructure; `AccessManager` removed for route roles[^3]. |
| 7.x | 2026 | Current stable (7.2.2); routing registered via `config.routes`, virtual-thread support[^2][^3]. |

## References

[^1]: Javalin README "Special thanks" credits Per Wendel (Spark) and the project's Spark heritage; project created by David Åse (tipsy). https://github.com/javalin/javalin
[^2]: GitHub API repository metadata for javalin/javalin, fetched 2026-07 (~8.3k stars, 644 forks, Apache-2.0, primary language Kotlin, last push 2026-07-01). https://github.com/javalin/javalin
[^3]: Javalin documentation and per-major migration guides. https://javalin.io/documentation

## Tags

kotlin, java, web-framework, http-server, websocket, rest-api, jetty, microservices, jvm, embedded-server
