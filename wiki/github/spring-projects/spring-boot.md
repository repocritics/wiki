# spring-projects/spring-boot

> Opinionated convention-over-configuration layer over the Spring Framework — embedded server, auto-configuration, executable JAR.

[GitHub repo](https://github.com/spring-projects/spring-boot) ·
[Official website](https://spring.io/projects/spring-boot) ·
[License: Apache-2.0](https://github.com/spring-projects/spring-boot/blob/main/LICENSE.txt)

## Overview

Spring Boot is a set of conventions, starter dependencies, and auto-configuration built on top of the Spring Framework. Spring Framework itself dates to 2003 and is a general-purpose dependency-injection and application container; by the early 2010s a typical Spring app required substantial XML and Java configuration before it did anything. Spring Boot 1.0 shipped in April 2014 to remove that ceremony: sensible defaults, embedded servers, and a single `java -jar` artifact[^1]. It did not replace Spring — every Boot app is a Spring app — it made the common case start in seconds.

The defining mechanism is **auto-configuration**: Boot inspects the classpath and existing beans and conditionally wires up infrastructure. If `spring-boot-starter-data-jpa` and an H2 driver are present, it configures a `DataSource`, an `EntityManagerFactory`, and a transaction manager without you writing any of it. This is the framework's greatest strength and its central tension: behavior is inferred from what is on the classpath, so adding or removing a dependency can silently change how the application wires itself, and diagnosing "why is this bean here / missing" is a recurring skill.

Spring Boot is the default choice for JVM backend services in enterprise Java — REST APIs, batch jobs, messaging consumers, data pipelines — and carries the corresponding weight: a large dependency graph, JVM warmup, and memory footprint that only recently gained a native-image escape hatch.

## Getting Started

Generate a project from Spring Initializr (`start.spring.io`) or the CLI, or add the parent POM / Gradle plugin manually. A minimal Maven `pom.xml` inherits `spring-boot-starter-parent` and adds `spring-boot-starter-web`. The complete application:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@SpringBootApplication
public class Example {

    @GetMapping("/")
    String home() {
        return "Hello World!";
    }

    public static void main(String[] args) {
        SpringApplication.run(Example.class, args);
    }
}
```

```bash
./mvnw spring-boot:run          # dev run
./mvnw package && java -jar target/*.jar   # executable fat JAR
```

`@SpringBootApplication` is a meta-annotation combining `@Configuration`, `@ComponentScan`, and `@EnableAutoConfiguration`. Configuration is externalized to `application.properties` / `application.yml`, overridable by environment variables and command-line args.

## Architecture / How It Works

**Auto-configuration** is driven by classes registered in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (pre-3.0 this was `spring.factories`). Each is gated by `@Conditional` annotations — `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, etc. At startup Boot evaluates these conditions against the actual classpath and bean set. `@ConditionalOnMissingBean` is what makes overriding work: define your own `DataSource` bean and the auto-configured one steps aside.

**Starters** (`spring-boot-starter-*`) are dependency aggregators — empty POMs that pull a curated, version-aligned set of transitive dependencies. The `spring-boot-dependencies` BOM pins versions across the entire ecosystem so that Jackson, Hibernate, Tomcat, and Micrometer versions are mutually compatible.

**Embedded servers.** Boot bundles a servlet container (Tomcat by default; Jetty or Undertow via dependency swap) or a reactive server (Netty for WebFlux). The container is a bean inside the app rather than the app being deployed into a container — inverting the traditional WAR-into-app-server model. WAR deployment is still supported but is the minority path.

**The executable JAR** is a nested-JAR format: dependencies remain as intact JARs inside `BOOT-INF/lib`, loaded by a custom `LaunchedClassLoader`. This is not a shaded/uber JAR, which avoids class-file merge conflicts.

**Two web stacks coexist.** Spring MVC (servlet, blocking, thread-per-request) and Spring WebFlux (reactive, Project Reactor, non-blocking) are separate and largely non-interchangeable programming models. Choosing WebFlux commits the whole request path to reactive types (`Mono`/`Flux`).

Spring Boot 3.0 (2022) moved the baseline to Java 17 and migrated from the `javax.*` to the `jakarta.*` namespace (Jakarta EE 9+), a breaking change that rippled through every dependency touching servlets, persistence, or validation[^2]. It also added first-class GraalVM **native image** support via ahead-of-time (AOT) processing.

## Production Notes

**Startup time and memory.** JVM warmup plus classpath scanning and condition evaluation makes cold start slower and idle memory heavier than lighter frameworks. This is the main reason Boot is a poor fit for scale-to-zero serverless without native compilation.

**Debugging auto-configuration.** When a bean is unexpectedly present or absent, run with `--debug` (or `debug=true`) to print the **condition evaluation report** showing which auto-configurations matched, did not match, and why. Learning to read this report is essential; guessing is not viable at scale.

**The jakarta namespace migration.** Upgrading across the 2.x → 3.0 boundary is not a version bump — any library referencing `javax.servlet`, `javax.persistence`, or `javax.validation` must have a jakarta-compatible release. Plan this as a project, not a dependency update.

**Native image (GraalVM) trade-offs.** AOT compilation yields sub-100ms startup and low memory, but build times are long (minutes), reflection/resource/proxy usage must be registered (Boot's AOT engine handles most, third-party libs may not), and dynamic behavior that works on the JVM can fail at native runtime. Treat native as a deliberate target, tested in CI, not a flag flipped late.

**Actuator exposure.** `spring-boot-starter-actuator` adds operational endpoints. By default only `health` and `info` are exposed over HTTP; exposing others (`env`, `heapdump`, `mappings`, `beans`) leaks internals and must be gated behind auth or a management port.

**Property precedence is deep.** Config comes from properties files, profile-specific files, environment variables, command-line args, and more, with a defined override order. Relaxed binding maps `MY_VAR` to `my.var`. Surprises usually trace to precedence, not to a missing value.

**Version alignment.** Let the BOM manage versions. Manually overriding a single transitive dependency (e.g. a newer Jackson) can break the compatibility guarantees the starter set relied on.

## When to Use / When Not

**Use when:**
- You're building JVM backend services and want the mainstream, heavily-documented, well-staffed default.
- You need the breadth of the Spring ecosystem — Data, Security, Batch, Integration, Cloud — pre-wired and version-aligned.
- Your team already knows Spring; Boot removes the configuration tax.
- You want one deployable artifact with an embedded server.

**Avoid when:**
- Cold-start latency and idle memory are primary constraints (serverless, dense multi-tenancy) and you can't invest in native image — Quarkus or Micronaut fit better.
- You want a small, transparent framework whose wiring you can hold in your head — auto-configuration is the opposite bet.
- You're writing a Kotlin-first, coroutine-centric service — Ktor is a more natural fit.
- The workload is a small script or single-purpose tool where the framework overhead dwarfs the logic.

## Alternatives

- quarkusio/quarkus — build-time DI and native-first design; use when startup time and container density matter more than ecosystem breadth.
- micronaut-projects/micronaut — compile-time dependency injection with no runtime reflection; use for low-memory serverless where reflection-heavy startup is the bottleneck.
- ktor/ktor — JetBrains' Kotlin/coroutine-native framework; use when the stack is Kotlin-first and you want a lighter surface than Spring.
- helidon-io/helidon — Oracle's MicroProfile/reactive framework; use in a Jakarta EE / MicroProfile-standardized shop.
- dropwizard/dropwizard — opinionated bundle of Jetty + Jersey + Jackson; use for a smaller, more explicit REST-service stack without Spring's magic.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014-04 | Initial release. Auto-config, starters, embedded servers, executable JAR[^1]. |
| 2.0 | 2018-03 | Spring Framework 5, WebFlux reactive stack, Micrometer metrics. |
| 2.3 | 2020-05 | Layered JARs, Cloud Native Buildpacks support. |
| 3.0 | 2022-11 | Java 17 baseline, jakarta namespace, GraalVM native image, AOT[^2]. |
| 3.2 | 2023-11 | JDK 21 virtual threads support, `RestClient`. |
| 3.5 | 2025-05 | Latest 3.x line[^3]. |

## References

[^1]: Spring blog, "Spring Boot 1.0 GA Released" — 2014-04-01. https://spring.io/blog/2014/04/01/spring-boot-1-0-ga-released
[^2]: Spring Boot 3.0 release notes / migration guide. https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Release-Notes
[^3]: Spring Boot project page and release history. https://spring.io/projects/spring-boot#learn

## Tags

java, spring, backend-framework, dependency-injection, jvm, rest-api, microservices, auto-configuration, embedded-server, enterprise-java, apache-2.0
