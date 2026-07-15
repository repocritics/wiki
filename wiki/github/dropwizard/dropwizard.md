# dropwizard/dropwizard

> Opinionated glue that bundles Jetty, Jersey, Jackson, and Metrics into a single fat-jar recipe for production JSON/REST services on the JVM.

[GitHub repo](https://github.com/dropwizard/dropwizard) ·
[Official website](https://www.dropwizard.io) ·
[License: Apache-2.0](https://github.com/dropwizard/dropwizard/blob/release/5.0.x/LICENSE)

## Overview

Dropwizard is a Java framework for building RESTful web services by pre-assembling a fixed set of mature libraries: Jetty for HTTP, Jersey (JAX-RS) for REST modeling, Jackson for JSON, Metrics for instrumentation, Logback for logging, Hibernate Validator for validation, and JDBI/Hibernate + Liquibase for persistence[^1]. It originated at Yammer, largely the work of Coda Hale, and was one of the first "opinionated bundle" JVM frameworks — predating and philosophically adjacent to Spring Boot. The Metrics library (dropwizard/metrics) spun out of it and is now used well beyond Dropwizard itself.

The defining stance is deliberate minimalism. Dropwizard is **not** a dependency-injection container. There is no `@Autowired`, no component scanning, no application context. You wire your objects together by hand inside a single `run()` method. This is the framework's central tension: it gives you a small, legible, boring application you can read top to bottom, at the cost of the ecosystem gravity and auto-configuration that make Spring Boot the default choice for large teams. Teams that value operational transparency over feature breadth pick Dropwizard; teams that want batteries plus a plugin for everything usually don't.

The project is still actively maintained but has settled into low-churn maturity — releases are infrequent and mostly track upstream library security updates and the JVM's Jakarta EE namespace transition rather than adding new surface area.

## Getting Started

Add the core artifact (Maven):

```xml
<dependency>
  <groupId>io.dropwizard</groupId>
  <artifactId>dropwizard-core</artifactId>
  <version>4.0.x</version>
</dependency>
```

A minimal application is a `Configuration`, an `Application`, and a JAX-RS resource:

```java
public class HelloConfig extends Configuration {
    @NotEmpty private String greeting = "Hello";
    public String getGreeting() { return greeting; }
}

@Path("/hello")
@Produces(MediaType.APPLICATION_JSON)
public class HelloResource {
    private final String greeting;
    public HelloResource(String greeting) { this.greeting = greeting; }

    @GET
    public Map<String, String> hello(@QueryParam("name") String name) {
        return Map.of("message", greeting + ", " + (name == null ? "world" : name));
    }
}

public class HelloApp extends Application<HelloConfig> {
    public static void main(String[] args) throws Exception {
        new HelloApp().run(args);
    }
    @Override
    public void run(HelloConfig config, Environment env) {
        env.jersey().register(new HelloResource(config.getGreeting()));  // manual wiring
    }
}
```

Configuration is YAML, deserialized into your `Configuration` subclass and validated:

```yaml
greeting: "Howdy"
server:
  applicationConnectors: [{ type: http, port: 8080 }]
  adminConnectors:       [{ type: http, port: 8081 }]
```

Build a fat jar with the Maven Shade plugin and run it with `java -jar app.jar server config.yml`.

## Architecture / How It Works

Dropwizard is a thin coordination layer over independent libraries rather than a runtime of its own. The moving parts:

- **`Application<C>`** — the entry point. `initialize()` registers `Bundle`s and `Command`s before config is read; `run()` receives the parsed config and the `Environment` and wires resources.
- **`Configuration`** — a POJO deserialized from YAML by Jackson and validated by Hibernate Validator. Nested `*Factory` objects (e.g. `DataSourceFactory`, `ServerFactory`, `LoggingFactory`) are polymorphic and construct the real components.
- **`Environment`** — the registration surface: `jersey()` for resources/providers, `healthChecks()`, `metrics()`, `lifecycle()` for managed objects, `admin()`.
- **Commands** — the CLI. `server` runs the app; `check` validates config; `db` runs Liquibase migrations. Custom subcommands are first-class.
- **Bundles** — reusable units that hook `initialize`/`run` (e.g. `MigrationsBundle`, `AssetsBundle`, `ViewBundle`). This is the extension mechanism in lieu of DI modules.

The runtime model is **classic blocking servlet-style I/O on a Jetty thread pool**, not reactive/event-loop. A request occupies a worker thread for its full duration. Jersey turns annotated resource classes into request handlers via reflection; Jackson (de)serializes representations; Hibernate Validator enforces `@Valid`/`@NotNull` constraints on request entities and returns 422 on failure.

A distinctive operational detail is the **two-port design**: the application listens on 8080 while an *admin connector* on 8081 exposes health checks (`/healthcheck`), Metrics (`/metrics`), thread dumps, and a ping endpoint. Health and instrumentation are structurally separated from application traffic — one of Dropwizard's better ideas, and one people forget to firewall.

## Production Notes

**The Jakarta namespace split is the single biggest thing to get right.** Dropwizard 3.x and 4.x were released in parallel and are the *same* framework differing only in Java EE namespace: 3.x stays on `javax.*` (Jakarta EE 8), 4.x moves to `jakarta.*` (Jakarta EE 9+)[^2]. Third-party integrations, servlet filters, and JAX-RS providers must match your line — mixing `javax.ws.rs` and `jakarta.ws.rs` artifacts fails at runtime in confusing ways. Plan the migration as a coordinated dependency sweep, not an incremental one.

**No DI is a feature until the app is large.** Hand-wiring in `run()` is readable at 10 resources and unwieldy at 100. Teams commonly bolt on Guice via `dropwizard-guicey` (community, not core) once construction boilerplate dominates — at which point some of Dropwizard's simplicity advantage over Spring Boot evaporates.

**Blocking I/O caps concurrency at the thread pool.** Because a request holds a Jetty worker for its lifetime, tail latency under load is governed by `server.maxThreads` and downstream call latency, not clever scheduling. High-fan-out or long-poll workloads need explicit pool sizing; there is no built-in reactive backpressure. Dropwizard is a poor fit for tens-of-thousands of concurrent slow connections.

**Dependency versions are pinned, for better and worse.** Dropwizard curates a coherent set of transitive versions (a BOM). This prevents Jetty/Jersey/Jackson mismatch, but also means you cannot casually bump Jackson ahead of what the release ships — you upgrade Dropwizard to move the stack.

**Logging is Logback-only and YAML-configured.** Standard `logback.xml` doesn't apply; logging is configured through the app YAML. Async appenders and request logging are built in. Redirecting `java.util.logging`/Log4j into the SLF4J bridge is automatic but occasionally surprises libraries that grab the root logger early.

**Operational upside.** First-class Metrics (timers/meters/histograms with reporter plugins for Graphite, Prometheus via community bridges), health checks wired into the admin port, and a single self-contained jar make Dropwizard genuinely pleasant to run and observe. This is where it still earns its keep.

## When to Use / When Not

**Use when:**
- You want a small, explicit JSON/REST service you can read end to end, with no magic.
- Metrics, health checks, and single-jar deployment matter more than framework breadth.
- The team prefers manual wiring and a curated, conflict-free library stack.
- You're building an operationally boring internal service or microservice.

**Avoid when:**
- You want dependency injection, auto-configuration, and a plugin for every datastore — Spring Boot's ecosystem wins.
- You need reactive/non-blocking I/O or GraalVM native-image fast startup (look at Quarkus/Micronaut).
- Your workload is many thousands of concurrent slow/streaming connections (blocking thread model).
- You want frequent releases and a large third-party integration marketplace.

## Alternatives

- spring-projects/spring-boot — use instead when you want dependency injection, auto-configuration, and the largest JVM integration ecosystem.
- quarkusio/quarkus — use when you need GraalVM native images, fast cold start, and reactive support for cloud/serverless.
- micronaut-projects/micronaut-core — use when you want compile-time DI and low memory without runtime reflection.
- javalin/javalin — use when Dropwizard still feels too heavy and you want a minimal Jetty-based web layer.
- ktorio/ktor — use when the team is Kotlin-first and wants coroutine-based non-blocking servers.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2011 | Initial release; grew out of Yammer internal tooling (Coda Hale)[^1]. |
| 1.0.0 | 2016-07 | First major release. Java 8, Jetty 9, Jersey 2, Jackson 2. |
| 2.0.0 | 2019-11 | JUnit 5 support, updated core libraries, Java 8/11. |
| 2.1.0 | 2022 | Jetty/Jersey updates on the `javax.*` line. |
| 3.0.0 | 2023-05 | Modernized `javax.*` (Jakarta EE 8) line[^2]. |
| 4.0.0 | 2023-05 | `jakarta.*` namespace (Jakarta EE 9+), released alongside 3.0[^2]. |
| 5.0.x | in dev | Current development line (default branch `release/5.0.x`). |

## References

[^1]: Dropwizard README and project site — component list and origin. https://www.dropwizard.io/
[^2]: Dropwizard release notes — 3.0 (`javax`) and 4.0 (`jakarta`) parallel release and namespace split. https://github.com/dropwizard/dropwizard/releases

## Tags

java, rest, jax-rs, jersey, jetty, web-framework, microservices, metrics, json, jvm, backend
