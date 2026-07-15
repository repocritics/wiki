# micronaut-projects/micronaut-core

> A JVM framework that resolves dependency injection and AOP at compile time instead of runtime, trading longer builds for fast startup, low memory, and first-class GraalVM native images.

[GitHub repo](https://github.com/micronaut-projects/micronaut-core) ·
[Official website](https://micronaut.io) ·
[License: Apache-2.0](https://github.com/micronaut-projects/micronaut-core/blob/5.2.x/LICENSE)

## Overview

Micronaut is a full-stack JVM application framework — dependency injection, AOP, an HTTP server and client, configuration, and a microservices toolkit (service discovery, distributed config, client-side load balancing). It was created by a team at Object Computing, Inc. that previously built the Grails framework, and 1.0 shipped in October 2018[^1]. It is now stewarded by the Micronaut Foundation. It supports Java, Kotlin, and Groovy from a single codebase.

The defining decision is that Micronaut does its dependency-injection and aspect-oriented-programming work **at compile time via annotation processors**, not at runtime via reflection and classpath scanning. Where Spring builds its bean graph by reflecting over classes at startup, Micronaut generates `BeanDefinition` classes during `javac` (or `kapt`/KSP for Kotlin) so the runtime just instantiates precomputed metadata. The direct consequences are startup measured in tens of milliseconds rather than seconds, a small heap, minimal reflection, no runtime bytecode generation, and — because reflection is minimized — clean support for GraalVM native images[^2]. This makes it a natural fit for serverless functions and CLI tools where cold-start time and memory dominate cost.

The tradeoff is paid at the edges. Compilation is slower and more fragile: the annotation processor must be wired into the build, generated code is harder to step through, and the ecosystem — integrations, tutorials, Stack Overflow answers — is far smaller than Spring's. Micronaut is a bet that startup/footprint/native-image matter more than breadth of third-party integrations.

## Getting Started

The recommended entry point is the Micronaut CLI (`mn`, installable via SDKMAN) or the web starter at [micronaut.io/launch](https://micronaut.io/launch):

```bash
sdk install micronaut
mn create-app com.example.demo --build gradle --lang java
cd demo
./gradlew run
```

A minimal HTTP controller:

```java
package com.example.demo;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

@Controller("/hello")
public class HelloController {

    @Get("/{name}")
    public String greet(String name) {
        return "Hello, " + name;
    }
}
```

The application class is a plain `main` that boots the context:

```java
import io.micronaut.runtime.Micronaut;

public class Application {
    public static void main(String[] args) {
        Micronaut.run(Application.class, args);
    }
}
```

## Architecture / How It Works

**Compile-time bean definitions.** The `io.micronaut.inject` annotation processor scans `@Singleton`, `@Bean`, `@Factory`, injection points, and AOP annotations at build time and emits one compiled `BeanDefinition` class per bean. At runtime the `ApplicationContext` reads these definitions directly — no classpath scanning, no reflective constructor discovery in the hot path. This is why forgetting to configure the processor produces a working compile but an empty context.

**AOP without runtime proxies.** Around-advice and introduction-advice (`@Around`, `@Introduction`) generate proxy subclasses at compile time. There is no CGLIB/ByteBuddy runtime enhancement, which is what keeps Micronaut compatible with GraalVM's closed-world assumption.

**HTTP on Netty.** The default server and client are built on Netty and are non-blocking. Handlers can return reactive types (`Publisher`, Reactor `Mono`/`Flux`, RxJava) or plain values; blocking work belongs on a separate executor, not the event loop. A Servlet-based runtime (`micronaut-servlet`, backed by Tomcat/Jetty/Undertow) exists for teams that need the blocking model.

**Reactive by default, Reactor-aligned.** Micronaut speaks Reactive Streams and, since 3.x, standardized on Project Reactor as the default reactive library while retaining RxJava interop.

**Companion projects.** The `micronaut-core` repo is the engine; most capability lives in sibling repos under `micronaut-projects` — Micronaut Data (compile-time query generation, the analog to Spring Data but again without runtime proxies), Micronaut Security, Micronaut Kafka, the cloud modules (AWS/GCP/Azure), and the GraalVM/AOT tooling. The Micronaut Platform BOM pins compatible versions across all of them.

## Production Notes

**The annotation processor is the number-one footgun.** In Gradle you must declare beans-generating dependencies under both `implementation` and `annotationProcessor` (Java) or `kapt`/`ksp` (Kotlin). Miss the processor scope and the code compiles but beans are never generated, surfacing as `NoSuchBeanException` at runtime rather than a build error. Kotlin users should prefer KSP over the older, slower kapt where a module supports it.

**Build time and IDE friction.** Because real work happens in `javac`, incremental builds are heavier than a reflection-based framework, and IDEs occasionally fail to pick up newly generated `BeanDefinition`s until a clean rebuild. Lombok interoperates but is order-sensitive — the Lombok processor must run before Micronaut's, which requires explicit ordering in some setups.

**GraalVM native image.** This is a headline use case, not a free one. `native-image` builds are slow (minutes) and memory-hungry, and any third-party library that uses reflection or dynamic proxies needs reachability metadata. Micronaut's own AOT tooling generates most of what its modules need, but bringing arbitrary JVM libraries into a native image can still require hand-written `reflect-config.json`. Budget CI accordingly.

**Don't block the event loop.** As with any Netty-based server, blocking calls (JDBC, `Thread.sleep`, synchronous HTTP) on a Netty worker thread will stall unrelated requests. Use `@ExecuteOn(TaskExecutors.BLOCKING)` or reactive drivers. Java 21 virtual threads ease this but do not eliminate the discipline.

**Ecosystem breadth.** The most common real-world blocker is not the framework but its surroundings: a Spring Boot starter usually exists for any given integration; a Micronaut equivalent sometimes does not, and you write the glue yourself. Evaluate integration coverage for your specific dependencies before committing.

**Upgrade pain — the jakarta namespace.** Micronaut 3.0 (2021) migrated from `javax.*` to `jakarta.*` for injection and validation annotations, matching the broader Jakarta EE move[^3]. Micronaut 4.0 (2023) raised the baseline to Java 17 and moved to Netty 4.1 / newer Jakarta APIs[^4]. Both were mechanical but touched imports across an entire codebase.

## When to Use / When Not

**Use when:**
- Cold-start latency and memory footprint are first-order costs — AWS Lambda and other serverless functions.
- You want GraalVM native-image executables with a framework that was designed for them rather than retrofitted.
- You are building fresh microservices or CLI tools and value fast startup and low RAM per instance.
- You have a polyglot JVM team (Java + Kotlin + Groovy) and want one framework.

**Avoid when:**
- You depend on the depth of the Spring ecosystem — its integrations, its documentation surface, its hiring pool.
- You are migrating a large existing Spring codebase; there is no drop-in path and idioms differ.
- Your team is not prepared to debug compile-time-generated code or maintain annotation-processor build config.
- Your workload is long-running and throughput-bound, where JIT warmup dwarfs startup time and Micronaut's core advantages barely register.

## Alternatives

- spring-projects/spring-boot — the incumbent; runtime DI, vastly larger ecosystem. Use when integration breadth and maturity outweigh startup/footprint, or when you are already invested in Spring.
- quarkusio/quarkus — the closest philosophical peer (build-time processing, GraalVM-first), Red Hat-backed. Use when you want MicroProfile/Jakarta EE alignment and a Red Hat support path.
- helidon-io/helidon — Oracle's MicroProfile + reactive framework. Use when MicroProfile standard compliance is the requirement.
- eclipse-vertx/vert.x — a lower-level reactive, event-driven toolkit without opinionated DI. Use when you want to assemble the stack yourself and control the reactor directly.
- ktorio/ktor — Kotlin-first, coroutine-based, minimal. Use for lightweight Kotlin-only services where you do not want annotation-processor machinery.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-10 | GA. Compile-time DI/AOP, Netty server, GraalVM support[^1]. |
| 2.0 | 2020-06 | Reactor/Kotlin improvements, expanded cloud modules. |
| 3.0 | 2021-08 | `javax.*` → `jakarta.*` migration; Reactor as default reactive lib[^3]. |
| 4.0 | 2023-07 | Java 17 baseline, Netty 4.1, newer Jakarta APIs, virtual-thread readiness[^4]. |
| 5.2.x | 2026 | Current default development branch as of this page's data fetch. |

## References

[^1]: Micronaut Foundation, "Micronaut 1.0 GA" announcement — October 2018. https://micronaut.io/2018/10/23/micronaut-1-0-ga-now-available/
[^2]: Micronaut user guide, "Ahead of Time (AOT) Compilation" / GraalVM. https://docs.micronaut.io/latest/guide/#graal
[^3]: Micronaut user guide, "Upgrading to Micronaut 3.0" (jakarta namespace). https://docs.micronaut.io/latest/guide/#upgrading
[^4]: Micronaut user guide, "Breaking Changes" — 4.0 (Java 17, Netty 4.1). https://docs.micronaut.io/latest/guide/#breaks

## Tags

java, jvm, dependency-injection, aop, microservices, serverless, graalvm, kotlin, netty, compile-time-di, framework
