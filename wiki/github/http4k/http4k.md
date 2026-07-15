# http4k/http4k

> A Kotlin HTTP toolkit built on "Server as a Function": an app is `(Request) -> Response`, and everything else is composition.

[GitHub repo](https://github.com/http4k/http4k) ·
[Official website](https://http4k.org) ·
[License: Apache-2.0](https://github.com/http4k/http4k/blob/master/LICENSE) (core; see Production Notes)

## Overview

http4k is a Kotlin HTTP library that models a web application as a plain function. Its two load-bearing type aliases are `HttpHandler = (Request) -> Response` and `Filter = (HttpHandler) -> HttpHandler`. A server, a client, a route, and a piece of middleware are all values of these types, composed with ordinary function composition. The design descends directly from Marius Eriksen's "Your Server as a Function" paper (Twitter's Finagle)[^1], and the project acknowledges Dan Bodart's *utterlyidle* as an ancestor[^2]. It was open-sourced in 2017[^3] by David Denton and Ivan Sanchez, extracted from production work.

The defining tension is minimalism versus convenience. http4k has no annotations, no reflection, no classpath scanning, no dependency-injection container, and a `http4k-core` with zero dependencies beyond the Kotlin stdlib. You wire the application yourself, explicitly, with constructors and function composition. In exchange you get an app that is trivially testable — because an `HttpHandler` is just a function, you call it directly in a unit test with no running server — and one that is symmetric: an HTTP *client* is also an `HttpHandler`, so the same abstractions and Filters apply to outbound calls. The cost is verbosity and manual wiring where Spring or Ktor would use convention or magic.

The platform ships as many add-on modules on a single synchronized version, pinned by a BOM. Recent releases have added `http4k-ai` (LLM adapters), `http4k-connect` (typed cloud/AI clients with in-memory fakes), and an open-core commercial tier — `http4k-pro` and http4k Enterprise (LTS) — which is worth understanding before adopting (see Production Notes).

## Getting Started

```kotlin
// build.gradle.kts
dependencies {
    implementation(platform("org.http4k:http4k-bom:6.55.0.0"))
    implementation("org.http4k:http4k-core")
    implementation("org.http4k:http4k-server-jetty") // a real server backend
}
```

```kotlin
import org.http4k.core.*
import org.http4k.core.Method.GET
import org.http4k.core.Status.OK
import org.http4k.routing.bind
import org.http4k.routing.routes
import org.http4k.server.Jetty
import org.http4k.server.asServer

val app: HttpHandler = routes(
    "/ping" bind GET to { Response(OK).body("pong") },
    "/echo" bind Method.POST to { req -> Response(OK).body(req.body) },
)

fun main() {
    app.asServer(Jetty(8080)).start()
}
```

Because the app is a function, the test needs no network:

```kotlin
val response = app(Request(GET, "/ping"))   // Response(OK, "pong")
```

## Architecture / How It Works

Everything is built from the two aliases:

- **HttpHandler** — `(Request) -> Response`. Synchronous and blocking by design; there is no `suspend` in the core signature.
- **Filter** — `(HttpHandler) -> HttpHandler`. Filters compose with `then`: `filterA.then(filterB).then(handler)`. Cross-cutting concerns (auth, logging, error mapping, tracing) are Filters, and the request/response chain is just nested function calls.

Other core pieces:

- **Immutable messages.** `Request` and `Response` are immutable; `.body(...)`, `.header(...)`, `.query(...)` return copies. No hidden mutable request state.
- **Lenses.** Typed, bidirectional accessors for extracting *and* injecting values: `Query.int().required("page")` reads from a request and can also build one. Failed extraction throws `LensFailure`, which you convert to a 400 with `ServerFilters.CatchLensFailure` — forgetting this Filter is a common source of 500s.
- **Routing.** `routes("/path" bind GET to handler)` produces a `RoutingHttpHandler`, itself an `HttpHandler`, so routers nest and mount freely.
- **Pluggable backends.** Servers (`SunHttp`, `Jetty`, `Netty`, `Undertow`, `Apache`, `Helidon`, `KtorCIO`) and clients (`JavaHttpClient`, `ApacheClient`, `OkHttp`, `JettyClient`) are interchangeable modules selected via `asServer(...)` / `asClient()`. Nothing in your app code depends on the choice.
- **Higher layers.** `http4k-contract` generates OpenAPI/Swagger from typed route specs; separate modules add WebSockets, SSE, multipart, templating, serverless adapters (AWS Lambda, GCP, Azure), and GraalVM native-image support. All share one version.

Because there is no framework runtime — no reflection, no annotation processing — startup is fast and memory footprint is small, which is why http4k is popular for serverless and GraalVM native binaries.

## Production Notes

- **`SunHttp` is not a production server.** It wraps the JDK's built-in `com.sun.net.httpserver` and is meant for tests and demos. Ship on `Jetty`, `Undertow`, or `Netty`. Deploying `SunHttp` by accident is the classic footgun.
- **Blocking, thread-per-request model.** The synchronous `HttpHandler` means each in-flight request holds a thread. This is a deliberate simplification, not an oversight, and it pairs well with JDK 21+ virtual threads (Loom) for high concurrency without a coroutine model. If you specifically want `suspend`/non-blocking end-to-end, Ktor is the better fit.
- **No DI container.** Dependency wiring is manual constructor injection. This stays clean for small/medium apps but requires discipline at scale; some teams bolt on a lightweight DI library or a hand-rolled composition root.
- **BOM version alignment.** Modules use a 4-part `MAJOR.MINOR.PATCH.BUILD` scheme and release very frequently (often several times a month). Always import `http4k-bom` and let it pin every module — mixing module versions manually is the most common build break.
- **Open-core licensing — read the LICENSE.** `http4k-core` and the standard modules are Apache-2.0, but content under `pro/` and the `org.http4k.pro` Maven group (Wiretap, X402/MPP, Verify, Hot Reload, pro MCP/A2A) is commercially licensed[^4]. GitHub's license detector reports `NOASSERTION` for the repo precisely because the `LICENSE` file carries a mixed preamble. Confirm which modules you depend on are Apache vs. paid before shipping.
- **Major upgrades are roughly biennial** (4.0 → 5.0 → 6.0) and remove deprecations and move packages; they need a planned migration pass, but the frequent minor releases are usually drop-in.
- **`LensFailure` surfaces as 500 without a catch Filter** — wire `CatchLensFailure` (or the contract module, which handles it) at the edge.

## When to Use / When Not

**Use when:**
- You want explicit, testable HTTP code with no magic, reflection, or annotations.
- You value the client/server symmetry (same Filters for inbound and outbound HTTP, in-memory fakes for tests).
- You're targeting serverless or GraalVM native and need fast startup / low memory.
- You practice TDD and want to test handlers as functions without spinning up a server.

**Avoid when:**
- You want a batteries-included framework with DI, ORM, and convention over configuration (Spring Boot).
- You need a fully non-blocking `suspend`-based stack or Kotlin Multiplatform clients (Ktor).
- Your team prefers annotation/compile-time-DI, native-first frameworks (Micronaut, Quarkus).
- You want to avoid any exposure to a commercial open-core tier in your dependency tree.

## Alternatives

- ktorio/ktor — coroutine-native, JetBrains-backed, multiplatform; use when you want `suspend` end-to-end or shared client code across platforms.
- spring-projects/spring-boot — full framework with DI and a vast ecosystem; use when convention and integrations matter more than minimalism.
- javalin/javalin — simple imperative servlet-based web framework; use when you want a lightweight Java/Kotlin API without the functional model.
- micronaut-projects/micronaut — compile-time DI, GraalVM-first; use when you want annotation-driven wiring with fast native startup.
- http4s/http4s — the Scala "server as a function" equivalent; use when your stack is Scala/Cats-Effect rather than Kotlin.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2017-03-23 | Open-sourced; "Server as a Function" for Kotlin[^3]. |
| 3.x | through 2020 | Long 3-part `3.MINOR.PATCH` line; contract/OpenAPI, serverless, WebSockets matured. |
| 4.0.0.0 | 2021-01-09 | Switched to 4-part `MAJOR.MINOR.PATCH.BUILD` versioning[^5]. |
| 5.0.0.0 | 2023-06-20 | Major line; package moves and deprecation cleanup[^5]. |
| 6.0.0.0 | 2025-02-13 | Current major; `http4k-ai`, `http4k-pro`, Enterprise LTS era[^5]. |
| 6.55.0.0 | 2026-07-04 | Latest release at time of writing[^5]. |

## References

[^1]: Marius Eriksen, "Your Server as a Function" (Twitter / Finagle), 2013. https://monkey.org/~marius/funsrv.pdf
[^2]: http4k README, Acknowledgments (utterlyidle, Dan Bodart; Ivan Moore). https://github.com/http4k/http4k#acknowledgments
[^3]: GitHub repository metadata, created 2017-03-23. https://github.com/http4k/http4k
[^4]: http4k commercial / pro licensing. https://www.http4k.org/commercial-license/
[^5]: http4k GitHub Releases. https://github.com/http4k/http4k/releases

## Tags

kotlin, jvm, http, functional-programming, server-as-a-function, http-server, http-client, microframework, testability, openapi, serverless
