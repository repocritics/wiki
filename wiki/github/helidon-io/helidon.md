# helidon-io/helidon

> Oracle's set of Java libraries for microservices — the framework that bet its 4.x rewrite entirely on virtual threads.

[GitHub repo](https://github.com/helidon-io/helidon) ·
[Official website](https://helidon.io) ·
[License: Apache-2.0](https://github.com/helidon-io/helidon/blob/main/LICENSE.txt)

## Overview

Helidon is a collection of Java libraries for building microservices, open-sourced by Oracle in 2018[^1]. It is not a monolithic framework in the Spring sense; it is a set of composable modules (web server, config, health, metrics, tracing, security, gRPC, DB client) that you assemble. It ships in two distinct programming models that share the same underlying libraries but present very different developer experiences.

**Helidon SE** is the low-level, no-magic model: explicit, functional-style builders, no dependency injection or annotation scanning, no reflection-heavy runtime. You wire routes and handlers in code. **Helidon MP** is an implementation of the Eclipse MicroProfile specification[^2] — CDI-based dependency injection, JAX-RS resources, declarative config and fault tolerance, familiar to anyone coming from Jakarta EE or Java EE. The two are not interchangeable at the API level; picking SE vs MP early is the most consequential decision on a Helidon project.

The defining event in Helidon's history is the 4.0 release (2023), which replaced the Netty-based reactive web server of 1.x–3.x with a new server codenamed **Níma**, written from scratch on Java 21 virtual threads (Project Loom)[^3]. This let the SE APIs move from asynchronous/reactive types back to plain blocking calls while keeping high throughput — thread-per-request programming without the reactive cognitive tax. The GitHub topics (`netty`, `reactive`) still reflect the pre-4.x architecture; they describe what Helidon was, not what 4.x is. As of 2026 the repository is actively developed (multiple pushes per week) with a comparatively modest ~3.8k stars — small next to Spring Boot or Quarkus, reflecting its position as Oracle's in-house framework rather than a broad community movement.

## Getting Started

Generate a project with the Helidon CLI or the Maven archetype (groupId `io.helidon`, no separate downloads):

```bash
# macOS/Linux CLI install
curl -O https://helidon.io/cli/latest/darwin/helidon
chmod +x ./helidon && sudo mv ./helidon /usr/local/bin/
helidon init            # interactive: SE or MP, base package, etc.
```

A minimal Helidon SE server (4.x, blocking handlers on virtual threads):

```java
import io.helidon.webserver.WebServer;
import io.helidon.webserver.http.HttpRouting;

public class Main {
    public static void main(String[] args) {
        WebServer.builder()
            .port(8080)
            .routing(Main::routing)
            .build()
            .start();
    }

    static void routing(HttpRouting.Builder routing) {
        routing.get("/greet", (req, res) -> res.send("Hello World!"));
    }
}
```

The MP equivalent is a JAX-RS resource with `@Path`/`@GET` and CDI beans, run via `io.helidon.microprofile.Main` — no `main` method wiring required.

## Architecture / How It Works

**Níma / WebServer (4.x).** The core of Helidon 4 is a from-scratch HTTP server built on virtual threads: each request is handled on its own virtual thread, so handler code blocks directly (JDBC, `HttpClient`, file I/O) without pinning a platform thread. This removes the `Single`/`Multi` reactive publisher types that pervaded Helidon SE 1.x–3.x. The tradeoff: virtual-thread performance depends on the JDK, and any code that holds a `synchronized` block or native call across an I/O point pins the carrier thread, silently defeating the model.

**Two models, one library set.** SE and MP are layered. MP is built on top of SE primitives plus CDI (Weld), JAX-RS (Jersey), and JSON-B/JSON-P. This means MP carries the classic Jakarta EE dependency weight and startup cost, while SE stays lean. Config, metrics (MicroProfile Metrics / Micrometer), health checks, OpenTelemetry tracing, and security are provided as shared modules usable from either model.

**GraalVM native image.** Both models support ahead-of-time compilation to a native binary for sub-100ms startup and low memory — a first-class target, though MP's reflection-heavy CDI/JAX-RS stack requires more native-image configuration than SE.

**Build model.** Helidon is Maven-first (3.8+), published under `io.helidon`. There is no Gradle-native tooling from the project; Gradle users consume the Maven artifacts. The main development branch tracks aggressive JDK adoption — the README on `main` states a future major (referred to as Helidon 27) requires JDK 26 to build and run[^4], continuing the project's pattern of pinning to recent Java LTS/feature releases.

## Production Notes

- **Java version floor is high and moves fast.** Helidon 4 requires Java 21; the development line targets even newer JDKs. This is deliberate (virtual threads need 21+) but rules Helidon out anywhere you are pinned to Java 17 or older. Upgrading Helidon often means upgrading the JDK in lockstep.
- **The 3.x → 4.x migration is a rewrite, not a bump.** SE code written against the reactive `Single`/`Multi` APIs does not run unchanged on 4.x; routing, handlers, and client code all changed shape. Oracle publishes an SE upgrade guide[^3], but budget real effort — this is the single biggest operational caveat.
- **Virtual-thread pinning.** The performance story assumes your dependencies are Loom-friendly. Legacy drivers using `synchronized` around blocking I/O, or heavy `ThreadLocal` use, can pin carrier threads and erase the throughput benefit. Profile under load before assuming thread-per-request scales.
- **SE vs MP is a one-way door in practice.** Migrating an established app from one model to the other is close to a rewrite. Choose based on team background: MP if you want annotation-driven Jakarta-EE-style development, SE if you want explicit control and minimal footprint.
- **Smaller ecosystem.** Fewer third-party integrations, Stack Overflow answers, and tutorials than Spring Boot or Quarkus. Support channels are the Helidon Slack and Stack Overflow `helidon` tag; the primary backer is Oracle.
- **Native image caveats.** MP native builds require maintaining reflection/resource configuration; expect iteration when adding libraries. SE native images are more straightforward.

## When to Use / When Not

**Use when:**
- You want virtual-thread-native, thread-per-request microservices on current Java (21+) without reactive-style code.
- Your team knows MicroProfile / Jakarta EE and wants a standards-based (MP) implementation.
- You want fine-grained, no-magic control over the runtime (SE) with minimal reflection and fast startup.
- You are already in an Oracle/OCI-leaning stack and want a vendor-supported Java microservice framework.

**Avoid when:**
- You are pinned to Java 17 or earlier — Helidon 4.x will not run.
- You need the largest possible ecosystem, hiring pool, and integration library — Spring Boot dominates there.
- You want build-time DI and the broadest GraalVM-native tooling with minimal config — Quarkus is more mature on that axis.
- You want a Gradle-first project — Helidon's tooling assumes Maven.

## Alternatives

- quarkusio/quarkus — Red Hat's Kubernetes-native Java framework; use instead when you want build-time DI, extensive GraalVM-native tooling, and a large extension catalog.
- spring-projects/spring-boot — use when you need the biggest ecosystem, hiring pool, and integration breadth and can accept a heavier runtime.
- micronaut-projects/micronaut-core — use when you want compile-time DI/AOP with low reflection and fast startup outside the MicroProfile model.
- eclipse-vertx/vert.x — use when you want an explicitly reactive, event-loop toolkit rather than thread-per-request.
- payara/Payara — use when you need a full Jakarta EE application server rather than an assembled microservice library set.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-09 | Open-sourced by Oracle as `oracle/helidon`[^1]. |
| 1.0 | 2019-02 | First GA. SE + MP (MicroProfile) on a Netty-based reactive server. |
| 2.0 | 2020-06 | New reactive WebClient, DB client, expanded MP support. |
| 3.0 | 2022-08 | Jakarta EE 9.1 (`jakarta.*` namespace), Java 17 baseline. |
| 4.0 | 2023-10 | Níma virtual-thread WebServer; SE APIs blocking again; Java 21 required[^3]. |

## References

[^1]: Oracle, "Announcing Helidon" — project open-sourced September 2018. https://medium.com/helidon
[^2]: Eclipse MicroProfile specification. https://microprofile.io/
[^3]: Helidon SE Upgrade Guide (Helidon 4, Níma / virtual threads). https://helidon.io/docs/v4/se/guides/upgrade_4x
[^4]: Helidon README, `main` branch (build/JDK requirements). https://github.com/helidon-io/helidon/blob/main/README.md

## Tags

java, microservices, microprofile, virtual-threads, project-loom, jakarta-ee, oracle, graalvm, reactive, web-framework, dependency-injection
