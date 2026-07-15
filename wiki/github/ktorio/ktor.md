# ktorio/ktor

> An unopinionated, coroutine-native HTTP toolkit for Kotlin — a library of composable pieces, not a batteries-included application framework.

[GitHub repo](https://github.com/ktorio/ktor) ·
[Official website](https://ktor.io) ·
[License: Apache-2.0](https://github.com/ktorio/ktor/blob/main/LICENSE)

## Overview

Ktor is a set of Kotlin libraries for building asynchronous servers and clients, built and maintained by JetBrains[^1]. Its defining choice is to be *unopinionated*: it ships an HTTP engine abstraction, a routing DSL, and a plugin (interception) mechanism, and leaves logging, serialization, templating, persistence, and dependency injection as opt-in choices. Where Spring Boot hands you a full application platform with conventions, Ktor hands you primitives and expects you to assemble the stack. That is simultaneously its main appeal (small surface, no reflection-heavy magic, fast startup) and its main cost (you wire more yourself, and there is no single "correct" project layout).

The engine is Kotlin coroutines from the ground up. Request handlers are `suspend` functions; concurrency is structured rather than thread-per-request, so a handler that awaits I/O does not block a platform thread. This makes Ktor a natural fit for high-fan-out microservices and API gateways where most time is spent waiting on downstream calls.

Ktor is also multiplatform on the client side: `ktor-client` runs on JVM, Android, Native (iOS/macOS/Linux), and JS/Wasm, which is why it is the default HTTP client in much Kotlin Multiplatform and Compose Multiplatform code. The server side is JVM-only.

## Getting Started

Server dependency (Gradle Kotlin DSL) plus an engine — Netty is the common default:

```kotlin
dependencies {
    implementation("io.ktor:ktor-server-core")
    implementation("io.ktor:ktor-server-netty")
}
```

```kotlin
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.application.*
import io.ktor.server.response.*
import io.ktor.server.routing.*

fun main() {
    embeddedServer(Netty, port = 8080) {
        routing {
            get("/") {
                call.respondText("Hello, world!")
            }
        }
    }.start(wait = true)
}
```

The [Ktor Gradle plugin](https://github.com/ktorio/ktor-build-plugins) supplies a BOM (so engine artifacts need no explicit version), fat-jar packaging, and run tasks. [start.ktor.io](https://start.ktor.io) is a project generator that pre-selects plugins.

## Architecture / How It Works

The core abstraction is the **pipeline**: an ordered list of interceptor phases through which an `ApplicationCall` (request + response) flows. Nearly everything user-facing — routing, content negotiation, authentication, compression, CORS — is a **plugin** that installs interceptors into these phases. `install(ContentNegotiation) { json() }` is the canonical shape[^2]. This is why Ktor feels small: the framework is mostly the pipeline machinery, and behavior is composed from plugins.

Routing is itself a plugin backed by a tree. Path segments, methods, and matchers form nodes; `get`, `post`, `route`, and nested blocks build the tree via a DSL. There is no annotation-based controller model and no compile-time route generation — routes are ordinary Kotlin code evaluated at startup.

**Engines** are pluggable and separate from application logic. Server engines include Netty, Jetty, Tomcat, a Servlet adapter, and **CIO** (Coroutine I/O), Ktor's own pure-Kotlin non-blocking engine. Client engines include CIO, OkHttp, Apache, Java's `HttpClient`, and platform engines (Darwin, Android, JS). You write against the same API and swap the engine artifact.

Serialization is not built in; `ContentNegotiation` plus `kotlinx.serialization` (or Jackson/Gson) converts request/response bodies. Dependency injection is likewise external by default — Koin is the common community pairing, though a first-party DI plugin was added in the 3.x line.

**Testing** is a first-class part of the design: `testApplication { }` spins the app in-process with no real socket, and `client.get("/")` exercises the full pipeline. It runs the same interceptors as production, which makes handler tests fast and realistic without mocking the framework.

## Production Notes

**Engine choice is a real decision.** Netty is the mature, widely deployed default. CIO has fewer dependencies and lower memory footprint but historically lagged Netty on edge cases (HTTP/2, some TLS scenarios). Benchmark with your own workload; do not assume CIO is faster because it is "pure Kotlin."

**The 1.x → 2.x migration was substantial.** Ktor 2.0 (2022) renamed "features" to "plugins", reworked the plugin-authoring API (`createApplicationPlugin`), and moved several packages[^3]. Code and third-party plugins written for 1.x do not compile unchanged. The 2.x → 3.x jump (2024) migrated internals to `kotlinx-io` and changed some byte-handling APIs[^4]; most application code survives, but custom plugins touching raw bytes may not.

**Coroutines leak your discipline.** Because handlers are `suspend` functions sharing structured concurrency, a blocking JDBC call or a `Thread.sleep` inside a handler stalls an event-loop thread and can starve unrelated requests. Wrap blocking work in `withContext(Dispatchers.IO)`. This is the most common Ktor performance footgun.

**Unopinionated means undocumented-by-convention.** There is no enforced project structure, so large Ktor codebases drift unless the team imposes its own module/plugin organization early. Onboarding engineers coming from Spring often expect autoconfiguration and controllers that are not there.

**Graceful shutdown and configuration.** `application.conf` (HOCON) or a YAML/programmatic config drives environment settings; wiring shutdown hooks, health endpoints, and connection draining is on you. Netty's connector settings (worker threads, request queue) are exposed but easy to leave at defaults that do not match load.

**Observability is assembled, not given.** Micrometer metrics, call logging, and OpenTelemetry exist as plugins/integrations, but you install and configure them — nothing is on by default.

## When to Use / When Not

**Use when:**
- You want a lightweight Kotlin HTTP service with fast startup and no reflection/classpath-scanning magic.
- Your workload is I/O-bound fan-out (gateways, BFFs, API aggregation) where coroutines shine.
- You need a multiplatform HTTP client (KMP, Compose Multiplatform, iOS/Android shared code).
- You prefer explicit wiring over convention and want to pick your own DI/ORM/serialization.

**Avoid when:**
- You want batteries included — security, data access, admin endpoints, autoconfiguration out of the box (Spring Boot / Quarkus / Micronaut fit better).
- Your team expects annotation-driven controllers and a large hiring pool familiar with the conventions.
- You have heavy blocking/JVM-ecosystem dependencies and no appetite for coroutine dispatch discipline.
- You need long-term API stability with rare breaking changes; Ktor's major versions have moved core APIs.

## Alternatives

- spring-projects/spring-boot — use instead when you want the full batteries-included platform and the largest JVM ecosystem, and accept heavier startup and reflection.
- quarkusio/quarkus — use instead when you want fast-startup/GraalVM-native JVM services with an opinionated, container-first stack.
- micronaut-projects/micronaut-core — use instead when you want compile-time DI/AOP (no runtime reflection) with a Spring-like programming model.
- http4k/http4k — use instead when you want "server as a function" with zero-magic, immutable HTTP and no coroutine model.
- javalin/javalin — use instead when you want a minimal Java/Kotlin web framework with a simpler, non-coroutine API.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-11 | First stable release; feature/pipeline model established[^1]. |
| 1.6 | 2021 | Late 1.x; broad plugin ecosystem, KMP client maturing. |
| 2.0 | 2022-04 | "Features" renamed to "plugins"; new plugin API; package moves[^3]. |
| 2.3 | 2023 | Continued 2.x hardening; more client/server plugins. |
| 3.0 | 2024-10 | Migration to `kotlinx-io`; performance and byte-handling changes[^4]. |
| 3.1.x | 2025–2026 | Current 3.x line (Gradle plugin `io.ktor.plugin` 3.1.x)[^5]. |

## References

[^1]: Ktor is an official JetBrains product; project site and history. https://ktor.io
[^2]: Ktor plugins and the interception/pipeline model. https://ktor.io/docs/server-plugins.html
[^3]: Ktor 2.0 migration guide (features → plugins, API changes). https://ktor.io/docs/migrating-2.html
[^4]: Ktor 3.0 migration guide (kotlinx-io migration). https://ktor.io/docs/migrating-3.html
[^5]: Ktor Gradle plugin, version referenced in the project README. https://github.com/ktorio/ktor-build-plugins

## Tags

kotlin, http-server, http-client, web-framework, coroutines, async, jvm, kotlin-multiplatform, microservices, jetbrains, unopinionated
