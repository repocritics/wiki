# spring-projects/spring-framework

> The dependency-injection container and application framework that underpins most of enterprise Java — the thing Spring Boot is built on top of.

[GitHub repo](https://github.com/spring-projects/spring-framework) ·
[Official website](https://spring.io/projects/spring-framework) ·
[License: Apache-2.0](https://github.com/spring-projects/spring-framework/blob/main/LICENSE.txt)

## Overview

Spring Framework is the core of the Spring ecosystem: an inversion-of-control (IoC) container plus a large set of integration modules (web MVC, reactive web, transactions, data access, messaging, testing) for building Java server applications[^1]. It grew out of Rod Johnson's 2002 book on J2EE design and shipped as 1.0 in 2004, positioned as a lighter alternative to the then-dominant EJB model. Two decades later it is the default substrate for enterprise Java, and its defining primitive — wiring objects ("beans") together through a container rather than `new` — is so pervasive that most Java developers encounter it indirectly.

The single most important thing to understand about this repository: **it is not Spring Boot.** Spring Boot (spring-projects/spring-boot) is a separate project that layers auto-configuration, an embedded server, and opinionated defaults on top of this framework. Nearly everyone starts a new application with Spring Boot, not with the raw framework. This page covers the underlying engine — the `ApplicationContext`, the bean lifecycle, AOP, Spring MVC/WebFlux — which Boot configures for you but does not replace.

The framework's central tension is **flexibility versus magic**. The container resolves dependencies, proxies beans for transactions and AOP, and scans the classpath at startup, mostly via reflection. This buys enormous configurability and a deep integration ecosystem, at the cost of startup time, memory footprint, opaque stack traces through generated proxies, and a learning curve dominated by understanding what the container did on your behalf.

## Getting Started

In practice you use Spring Boot, which pulls the framework in transitively. With Gradle:

```groovy
// build.gradle — Spring Boot brings spring-framework in transitively
plugins { id 'org.springframework.boot' version '3.4.0' }
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

```java
// A component and a REST controller — the container wires GreetingService in.
@Service
class GreetingService {
    String greet(String name) { return "Hello, " + name; }
}

@RestController
class GreetingController {
    private final GreetingService service;

    GreetingController(GreetingService service) {  // constructor injection
        this.service = service;
    }

    @GetMapping("/greet/{name}")
    String greet(@PathVariable String name) {
        return service.greet(name);
    }
}
```

Using the framework directly (no Boot) means constructing an `ApplicationContext` yourself and registering configuration classes — viable for libraries and non-web apps, rare for services.

## Architecture / How It Works

The core is the **IoC container**. `BeanFactory` is the base contract; `ApplicationContext` is the full-featured implementation applications actually use. At startup the container:

1. **Reads bean definitions** — from `@Configuration` classes and `@Bean` methods, classpath component scanning (`@Component`, `@Service`, `@Repository`, `@Controller`), or legacy XML.
2. **Instantiates and wires beans** — resolving constructor/field/setter dependencies (`@Autowired`), respecting scopes (singleton by default, plus prototype, request, session).
3. **Applies `BeanPostProcessor`s** — the extension point that implements much of Spring's behavior, including AOP proxy creation.
4. **Publishes lifecycle callbacks** — `@PostConstruct`, `InitializingBean`, `ApplicationListener` events.

**AOP is proxy-based.** Cross-cutting concerns (`@Transactional`, `@Async`, `@Cacheable`, custom aspects) are implemented by wrapping beans in a proxy — a JDK dynamic proxy when the bean implements an interface, otherwise a CGLIB subclass. This is why a self-invocation (a method calling another `@Transactional` method on `this`) silently bypasses the advice: the call never crosses the proxy boundary. This is the single most common Spring correctness bug.

**Two web stacks coexist.** Spring MVC is the servlet (blocking, thread-per-request) stack built on the `DispatcherServlet`. Spring WebFlux, added in 5.0, is a reactive non-blocking stack built on Project Reactor (`Mono`/`Flux`)[^2]. They share annotations but are separate runtimes; you pick one per application and do not mix them casually.

**Configuration evolved in layers** and all layers still work: XML (2004), annotation-driven (`@Autowired`, 2.5/3.0), and Java `@Configuration` (3.0, now dominant). Old XML-configured apps still run unchanged, which is a compatibility strength and a source of "there are five ways to do this" confusion.

## Production Notes

**The javax → jakarta migration (Spring 6.0) is the hardest upgrade in the framework's history.** Spring 6.0 (November 2022) moved to a Java 17 baseline and Jakarta EE 9+, which renamed every `javax.*` EE package (`javax.servlet`, `javax.persistence`, `javax.validation`) to `jakarta.*`[^3]. Upgrading a Spring 5 app means every dependency in the tree must also have published a jakarta-namespace release. Teams on old libraries were blocked for months. Budget this as a real project, not a version bump.

**Startup time and reflection.** Classpath scanning plus reflective bean instantiation makes cold start noticeable — hundreds of milliseconds to several seconds for large apps. This is fine for long-lived servers, painful for serverless/functions. Spring 6.0 added **AOT processing and GraalVM native-image support**[^3]: an ahead-of-time build step generates the reflection metadata and proxy classes so a native binary starts in tens of milliseconds. The catch is that native image forecloses runtime dynamism — anything using reflection, dynamic proxies, or resources must be registered as hints, and not all libraries provide them.

**Proxy footguns beyond self-invocation:** `@Transactional` on a `private` or `final` method is silently ignored (CGLIB cannot override it); `final` classes cannot be proxied at all without interface-based proxies; and injecting a prototype-scoped bean into a singleton captures a single instance unless you use a scoped proxy or `ObjectProvider`.

**Version support windows are short for the OSS line.** Open-source maintenance for a given feature branch is roughly a year after the next major/minor; extended support requires a commercial VMware/Broadcom subscription. Staying on a supported Spring version is effectively mandatory for security patches, which forces a fairly aggressive upgrade cadence.

**Testing is a genuine strength.** `spring-test` provides a context cache so integration tests reuse the same `ApplicationContext` across classes, plus `MockMvc`/`WebTestClient` for controller testing and `@MockBean` (Boot) for slice tests. Misconfiguring context caching (too many distinct configurations) silently multiplies test suite time.

## When to Use / When Not

**Use when:**
- You're building server-side Java and want the mainstream, deeply-supported ecosystem (data, security, messaging, cloud all integrate first-class).
- You want the largest hiring pool and documentation surface in enterprise Java.
- You need mature transaction management, declarative security, and a huge library of integrations.

**Avoid when:**
- Startup latency or memory is the primary constraint and you can't invest in native image — Quarkus and Micronaut do compile-time DI and start faster by default.
- You only need dependency injection, not a framework — a lightweight DI library is far less to reason about.
- You're not on the JVM. Spring is Java/Kotlin-only; there is no equivalent runtime elsewhere.

## Alternatives

- quarkusio/quarkus — build-time DI and native-first; much faster startup, smaller footprint. Use instead when serverless/Kubernetes density matters more than ecosystem breadth.
- micronaut-projects/micronaut-core — compile-time DI with no runtime reflection. Use when you want Spring-like ergonomics without the reflection/startup cost.
- google/guice — a DI container only, no web/data framework. Use when you want injection wiring and nothing else.
- helidon-io/helidon — Oracle's lightweight MicroProfile-based framework. Use instead in a Jakarta EE / MicroProfile-standard shop.
- google/dagger — compile-time DI (heavily used on Android). Use when you want zero-reflection injection and are not building a Spring-style server.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2004-03 | First release; IoC container as a lighter alternative to EJB[^1]. |
| 2.0 | 2006-10 | Extensible XML config, AspectJ integration. |
| 3.0 | 2009-12 | Java-based `@Configuration`, REST support in Spring MVC, SpEL. |
| 4.0 | 2013-12 | Java 8 support, WebSocket, `@RestController`. |
| 5.0 | 2017-09 | Reactive stack (WebFlux) on Project Reactor; Java 8 baseline[^2]. |
| 6.0 | 2022-11 | Java 17 baseline, Jakarta EE 9+ (`javax`→`jakarta`), AOT + GraalVM native[^3]. |
| 6.1 | 2023-11 | JDK 21 support incl. virtual threads; new `RestClient`. |
| 6.2 | 2024-11 | Bean-container and validation refinements; latest feature line. |

## References

[^1]: Spring Framework reference documentation, Overview. https://docs.spring.io/spring-framework/reference/overview.html
[^2]: Spring blog, "Reactive Spring" / Spring Framework 5.0 release. https://spring.io/blog/2017/09/28/spring-framework-5-0-goes-ga
[^3]: Spring blog, "Spring Framework 6.0 goes GA" — Jakarta EE 9+, Java 17, AOT/native. https://spring.io/blog/2022/11/16/spring-framework-6-0-goes-ga

## Tags

java, jvm, dependency-injection, ioc-container, web-framework, enterprise, spring, aop, reactive, backend, kotlin
