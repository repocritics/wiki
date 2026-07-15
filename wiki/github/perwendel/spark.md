# perwendel/spark

> A Sinatra-inspired micro web framework for Java 8 — routes as lambdas, embedded Jetty, no configuration.

[GitHub repo](https://github.com/perwendel/spark) ·
[Official website](http://sparkjava.com) ·
[License: Apache-2.0](https://github.com/perwendel/spark/blob/master/LICENSE)

## Overview

Spark (published as `com.sparkjava:spark-core`, and often called "Spark Java" to disambiguate) is a micro web framework in the Sinatra lineage. You declare routes as lambdas — `get("/hello", (req, res) -> "Hello World")` — and an embedded Jetty server starts on first route registration. There is no annotation scanning, no application context, no XML, and no code generation. A working HTTP service is a `main` method and a handful of static imports.

It is emphatically **not** Apache Spark, the distributed data-processing engine. The name collision predates Apache Spark's rise and is a persistent source of search, dependency, and Stack Overflow confusion; the Maven groupId `com.sparkjava` and the `spark-java` SO tag are the disambiguators.

The defining tension is age versus scope. Spark was among the first frameworks to make Java 8 lambdas feel idiomatic for web routing, and for a stretch (roughly 2015–2018) it was the default answer to "I want Flask/Sinatra for the JVM." But the project is now effectively dormant: the default branch's last push was October 2023[^1], the newest release is 2.9.4, and 262 issues sit open. What was once "minimal and fresh" is now "minimal and unmaintained" — a meaningful distinction for anything internet-facing.

## Getting Started

```xml
<dependency>
    <groupId>com.sparkjava</groupId>
    <artifactId>spark-core</artifactId>
    <version>2.9.4</version>
</dependency>
```

```java
import static spark.Spark.*;

public class HelloWorld {
    public static void main(String[] args) {
        port(4567);                        // default is already 4567
        get("/hello", (req, res) -> "Hello World!");
        get("/users/:name", (req, res) -> "Selected user: " + req.params(":name"));

        post("/echo", (req, res) -> req.body());
    }
}
```

Run `main`, then hit `http://localhost:4567/hello`. There is no build step, servlet container, or `web.xml` — the embedded Jetty instance is spun up automatically in a background thread on first route registration.

## Architecture / How It Works

The public API is a facade of static methods on `spark.Spark`, imported with `import static spark.Spark.*`. Under the hood these delegate to a singleton `Service`. The core concepts are small:

- **Routes** — `get/post/put/delete/...` take a path pattern and a `Route` lambda `(Request, Response) -> Object`. The returned object is written as the body; a `ResponseTransformer` (e.g. a Gson wrapper) can serialize POJOs to JSON.
- **Path params & splats** — `:name` captures a segment (`req.params(":name")`); `*` is a wildcard splat. Routes can also be constrained by accept type: `get("/x", "application/json", handler)`.
- **Filters** — `before`, `after`, and `afterAfter` run around matched routes; `afterAfter` runs even when a handler throws. `halt(status, body)` short-circuits the chain.
- **Views** — `ModelAndView` + a `TemplateEngine` implementation integrate FreeMarker, Velocity, Mustache, Thymeleaf, etc. via optional `spark-template-*` modules.
- **Static files** — `staticFileLocation("/public")` serves classpath resources; `externalStaticFileLocation` serves from the filesystem.

The server layer is **embedded Jetty** (9.x), thread-per-request. Spark deliberately hides Jetty behind its own `Request`/`Response` wrappers, though the underlying `HttpServletRequest`/`HttpServletResponse` remain reachable via `raw()`.

The most important architectural fact is the **global static singleton**. The `spark.Spark.*` DSL mutates one shared `Service`. This is what makes the hello-world so short, but it also means route registration is process-global and order-sensitive, and it complicates testing and running multiple servers in one JVM. The escape hatch is `Service.ignite()`, which returns an isolated, instance-scoped server you can configure with its own port and lifecycle (`.stop()`, `.awaitInitialization()`). A Kotlin DSL lives in the separate `perwendel/spark-kotlin` repo.

## Production Notes

- **Maintenance status is the headline caveat.** No release since 2.9.4 and no default-branch commits since late 2023[^1]. Bugs and CVEs in the dependency tree will not be patched upstream. Treat Spark as frozen: fine for internal tools, risky for new public services.
- **Transitive Jetty exposure.** Spark pins an older Jetty 9.x line. Jetty has had CVEs over the years; since Spark won't bump it, you inherit whatever the pinned version ships. Audit `mvn dependency:tree` and consider forcing a patched Jetty where compatible, or pinning at the reverse proxy.
- **Global state and tests.** Because the default DSL is a process-wide singleton, tests that register routes leak into each other and ports collide. Use `Service.ignite()` per test, call `awaitInitialization()` before asserting, and `stop()` in teardown. Parallel test execution against the static API does not work cleanly.
- **No batteries.** No JSON, no DI, no validation, no ORM, no config system. You wire in Gson/Jackson via `ResponseTransformer`, and there is no dependency-injection story — dependencies are captured in lambda closures or passed by hand. This is by design, but scaling past a few dozen routes tends to produce ad-hoc structure.
- **Thread-per-request, no async.** There is no reactive or non-blocking model and no HTTP/2 story. Concurrency is bounded by Jetty's thread pool; long-blocking handlers need pool tuning.
- **JDK compatibility.** Java 8 is the floor. It runs on newer JDKs, but the old embedded Jetty can surface module-system and TLS-default friction on very recent LTS releases; verify on your target JDK rather than assuming.
- **Exception mapping.** Uncaught exceptions default to a 500 with a generic body; register `exception(MyException.class, handler)` explicitly, and note that `notFound`/`internalServerError` custom pages are configured through the DSL, not conventions.

## When to Use / When Not

**Use when:**
- You want a tiny HTTP endpoint embedded in a larger app, a prototype, or a teaching example with near-zero ceremony.
- You value reading the whole framework in an afternoon over ecosystem breadth.
- The service is internal, short-lived, or behind a hardened proxy where the frozen dependency tree is acceptable.

**Avoid when:**
- You are starting a new production, internet-facing service in 2026 — the lack of maintenance and unpatched transitive deps is a real liability.
- You need async/reactive, HTTP/2, WebSockets at scale, DI, or a maintained plugin ecosystem.
- You want long-term support, security response, or an active community roadmap.

## Alternatives

- javalin/javalin — the spiritual successor, started by a former Spark contributor; near-identical lightweight DSL on Jetty but actively maintained, with WebSocket/HTTP/2 and a plugin ecosystem. Use this instead of Spark for almost any new lightweight-JVM service.
- spring-projects/spring-boot — use when you outgrow the micro model and want DI, data, security, and a vast ecosystem, accepting heavier startup and configuration.
- quarkusio/quarkus — use when startup time, memory, and GraalVM native images matter (serverless, containers).
- micronaut-projects/micronaut-core — use when you want compile-time DI/AOP and fast startup without reflection.
- ktor — use when the service is Kotlin-first and you want an idiomatic, coroutine-based async framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-05 | Repo created; Sinatra-style routing for pre-Java-8 (anonymous classes)[^1]. |
| 2.0 | ~2015 | Rewrite around Java 8 lambdas — the form the framework is known for.[^2] |
| 2.5–2.7 | 2016–2018 | Peak adoption; template engines, static file handling, WebSocket support added. |
| 2.9.x | 2020–2022 | Maintenance line; 2.9.4 is the newest published artifact[^3]. |
| (dormant) | 2023-10 | Last default-branch push; no releases since[^1]. |

## References

[^1]: perwendel/spark repository metadata (created 2011-05-05, default branch `master` last pushed 2023-10-08; 9,655 stars, 1,567 forks, 262 open issues, Apache-2.0), via GitHub API, retrieved 2026-07. https://github.com/perwendel/spark
[^2]: Spark documentation — routes, filters, and the Java 8 lambda DSL. http://sparkjava.com/documentation
[^3]: `com.sparkjava:spark-core` on Maven Central (latest published 2.9.4). https://mvnrepository.com/artifact/com.sparkjava/spark-core

## Tags

java, web-framework, microframework, rest-api, jetty, sinatra-inspired, jvm, routing, embedded-server, unmaintained
