# eclipse-vertx/vert.x

> A Netty-based toolkit for writing asynchronous, event-loop-driven applications on the JVM — not a framework, a set of libraries you compose yourself.

[GitHub repo](https://github.com/eclipse-vertx/vert.x) ·
[Official website](https://vertx.io) ·
[License: EPL-2.0 OR Apache-2.0](https://github.com/eclipse-vertx/vert.x/blob/master/LICENSE.txt)

## Overview

Vert.x is a toolkit for building reactive, non-blocking applications on the JVM, built directly on top of Netty[^1]. It began around 2011–2012 as "Node.x" (later vert.x), created by Tim Fox at VMware as a polyglot answer to Node.js's event-loop model; governance later moved to the Eclipse Foundation, where the project lives today[^2]. The word "toolkit" is load-bearing: unlike Spring, Vert.x imposes no application structure, no dependency-injection container, and no required lifecycle. You wire together `vertx-core` plus the pieces you need (web, JDBC/SQL clients, event bus, gRPC, auth) and control the composition yourself.

The defining model is the **multi-reactor pattern**: rather than one event loop (Node.js), Vert.x runs several — by default `2 × CPU cores` — and pins each unit of deployment to one. The uncompromising rule that follows is **never block the event loop**. Any blocking call (JDBC, file I/O on a blocking API, a slow computation) stalls every connection served by that loop. This is the framework's central tension: it delivers very high throughput per thread, but pushes the entire cost of avoiding blocking onto the developer, who must reach for worker threads, `executeBlocking`, or fully async clients for everything.

Vert.x is most used as the substrate under higher-level stacks — most notably Quarkus, whose reactive core is Vert.x[^3] — and directly for API gateways, high-connection HTTP/TCP services, event-driven microservices, and the event-bus-based messaging that ties them together.

## Getting Started

Add the core dependency (Maven):

```xml
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-core</artifactId>
  <version>5.0.0</version>
</dependency>
```

A minimal HTTP server as a verticle, using the Future-based API:

```java
import io.vertx.core.*;

public class Server extends VerticleBase {
  @Override
  public Future<?> start() {
    return vertx.createHttpServer()
      .requestHandler(req -> req.response()
        .putHeader("content-type", "text/plain")
        .end("Hello from Vert.x"))
      .listen(8080);   // returns a Future; non-blocking
  }

  public static void main(String[] args) {
    Vertx.vertx().deployVerticle(new Server());
  }
}
```

The `listen` call returns immediately; the returned `Future` completes when the port is bound. Nothing here occupies a thread while waiting for connections.

## Architecture / How It Works

**Event loops and verticles.** A `Vertx` instance owns a pool of event-loop threads. A **verticle** is the unit of deployment — a component with `start`/`stop` lifecycle that Vert.x assigns to a single event loop. Standard verticles run on an event loop and must never block; **worker verticles** run on a separate worker pool and may block. Scaling out is done by deploying many instances of the same verticle across the available loops (`setInstances(n)`).

**The event bus.** Vert.x's signature abstraction is an address-based message bus. Handlers register on string addresses; senders use point-to-point (`send`), publish/subscribe (`publish`), or request/reply. With a cluster manager (Hazelcast is the default; Infinispan, Ignite, and Zookeeper are supported) the bus spans a cluster transparently, so a message sent on an address can be handled by a verticle on another node[^4]. Messages are typed via codecs; the default assumes `String`/`JsonObject`/buffers unless you register a custom codec.

**Async API evolution.** This is where most of the project's history lives, and where migration pain concentrates:
- Vert.x 2/3 exposed **callback**-style APIs (`handler -> AsyncResult`), which produced the "callback hell" the ecosystem is known for.
- Vert.x 3 added **RxJava** bindings as an escape hatch.
- Vert.x 4 (2021) made **`Future`/`Promise`** first-class and introduced distributed tracing and a redesigned metrics/context model[^5].
- Vert.x 5 (2025) removed most callback-based method variants, adopted the Java Platform Module System (JPMS), and consolidated on Futures[^6].

On top of the core `Future`, four idioms coexist: raw callbacks (legacy), Vert.x `Future`, **RxJava 3 / SmallRye Mutiny** reactive types, and **Kotlin coroutines**. Quarkus standardizes on Mutiny; plain Vert.x users most often use `Future` composition (`.compose()`, `.map()`, `Future.all()`).

**Netty underneath.** HTTP/1.1, HTTP/2, WebSocket, TCP, UDP, and DNS all delegate to Netty. Native transports (epoll on Linux, io_uring, kqueue on BSD/macOS) are optional and enabled by adding the matching `netty-transport-native-*` artifact; without them Vert.x falls back to Java NIO.

## Production Notes

**Blocking the event loop is the #1 production incident.** Vert.x ships a **blocked-thread checker** that logs a warning (default: a thread blocked > 2s) with a stack trace. Treat those warnings as bugs, not noise — a single synchronous JDBC call or `Thread.sleep` in a handler degrades every client on that loop. Use the reactive SQL clients (`vertx-pg-client`, `vertx-mysql-client`) instead of JDBC where possible; when you must use a blocking library, wrap it in `vertx.executeBlocking(...)` or a worker verticle.

**Thread-affinity semantics.** A verticle's handlers always run on the same event-loop thread, which lets you write single-threaded-style code without locks — but it also means work does not automatically parallelize. To use all cores you must deploy multiple instances. Objects shared across event loops (e.g. via the event bus) must be immutable or copied; the bus copies by default for local `JsonObject`/buffer messages but not for arbitrary Java objects sent with a reference codec.

**Clustering is heavier than it looks.** The clustered event bus inherits the operational characteristics of its cluster manager. Hazelcast split-brain, member-discovery configuration, and serialization mismatches are common causes of "messages silently not delivered." Test failover explicitly; the bus does not guarantee delivery (it is fire-and-forget for `send`/`publish`).

**Upgrade pains.**
- 3.x → 4.x was a large API migration: callbacks-plus-`AsyncResult` gave way to `Future`, and package/metric APIs moved. Expect real code changes, not a version bump.
- 4.x → 5.x drops many callback overloads and introduces JPMS module boundaries, which can surface split-package and `Automatic-Module-Name` conflicts in large classpaths. Vert.x 5 also raises the minimum JDK.
- Polyglot language bindings (JavaScript/Nashorn, Ruby, Groovy, Scala) have been progressively de-emphasized; new work is Java-, Kotlin-, and Mutiny-centric. Do not assume the polyglot story from older tutorials still holds.

**Observability.** Metrics (Micrometer/Dropwizard) and OpenTelemetry/Zipkin tracing are opt-in modules, not on by default. Wire them early — retrofitting context propagation across the event bus after the fact is tedious.

## When to Use / When Not

**Use when:**
- You need very high connection counts / throughput per JVM (API gateways, proxies, streaming, WebSocket fan-out).
- You want async I/O without adopting a full framework's conventions — a library you compose, not a container that owns your app.
- You are building event-driven microservices and want a built-in (optionally clustered) message bus.
- You are already on Quarkus and want to drop to its reactive engine directly.

**Avoid when:**
- Your team is used to synchronous, blocking, transaction-script code (Spring MVC + JDBC): the "never block the loop" discipline is a real cost and a common source of production bugs.
- Your workload is dominated by blocking dependencies with no async client — you will spend most time on worker pools and lose the model's advantage.
- You want batteries-included conventions, DI, and a large opinionated ecosystem out of the box (choose Spring Boot).
- The app is simple CRUD with modest concurrency — the async tax buys you little.

## Alternatives

- spring-projects/spring-boot — opinionated, DI-driven, synchronous-first (with WebFlux as a reactive option); choose it when you want convention and ecosystem over raw async control.
- quarkusio/quarkus — built on Vert.x; use it when you want the same reactive core plus a full framework, DI, and native-image/GraalVM tooling.
- reactor/reactor-netty — the Netty-based reactive foundation under Spring WebFlux; use it when you want reactive streams tightly integrated with the Spring/Reactor stack.
- netty/netty — go one level down to raw Netty when you need protocol-level control and don't want Vert.x's verticle/event-bus model.
- ktorio/ktor — Kotlin-first async server toolkit built on coroutines; use it when the team is Kotlin-native and prefers coroutines over Futures/Mutiny.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2012 | Origins as "Node.x"/vert.x at VMware; polyglot event-loop toolkit[^2]. |
| 2.x | 2013–2014 | Move toward Eclipse Foundation governance; module system. |
| 3.0 | 2015-06 | Major rewrite; RxJava bindings; reactive SQL/web components[^1]. |
| 4.0 | 2021-02 | `Future`-first API, distributed tracing, redesigned metrics/context[^5]. |
| 5.0 | 2025 | Callback overloads removed, JPMS modules, raised minimum JDK[^6]. |

## References

[^1]: Vert.x website — "Vert.x is a tool-kit for building reactive applications on the JVM." https://vertx.io
[^2]: Eclipse Vert.x project overview, Eclipse Foundation. https://projects.eclipse.org/projects/rt.vertx
[^3]: Quarkus — "Quarkus Reactive Architecture" (built on Vert.x). https://quarkus.io/guides/quarkus-reactive-architecture
[^4]: Vert.x Core manual — "The Event Bus" and clustering. https://vertx.io/docs/vertx-core/java/#event_bus
[^5]: Vert.x 4 announcement / migration guide. https://vertx.io/blog/eclipse-vert-x-4-0-0-released/
[^6]: Vert.x 5 release notes and migration guide. https://vertx.io/docs/5.0.0/
[^7]: Repository metadata (stars, forks, license, activity) via GitHub API, fetched 2026-07-15. https://github.com/eclipse-vertx/vert.x

## Tags

java, jvm, reactive, event-loop, non-blocking, netty, async, event-bus, microservices, http2, toolkit, concurrency
