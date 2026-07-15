# junit-team/junit-framework

> The default testing framework for Java and the JVM — a launcher platform plus a programming model, not a single library.

[GitHub repo](https://github.com/junit-team/junit-framework) ·
[Official website](https://junit.org) ·
[License: EPL-2.0](https://github.com/junit-team/junit-framework/blob/main/LICENSE.md)

## Overview

This repository is the home of what most developers still call "JUnit 5": the JUnit Platform, JUnit Jupiter, and JUnit Vintage. It was created in 2015 under the name `junit5` and renamed to `junit-framework` after the project began shipping the JUnit 6 line from the same codebase[^1]. JUnit 5.0.0 reached general availability on 2017-09-10[^2], a ground-up rewrite of the aging JUnit 4 architecture funded partly through the 2015 "JUnit Lambda" crowdfunding campaign.

The defining decision of JUnit 5 was to split the monolith. JUnit 4 was one jar in which the only real extension point was the `@RunWith` runner — and a test class could have exactly one runner, so Spring, Mockito, and parameterized tests all competed for the same slot. JUnit 5 separates concerns into three pieces: the **Platform** (a `TestEngine` SPI and a `Launcher` API that IDEs and build tools target), **Jupiter** (the new annotations and extension model that test authors write against), and **Vintage** (a `TestEngine` that runs existing JUnit 3/4 tests on the new Platform). This is the framework's central tension: the added indirection buys a composable extension model and multi-framework support, at the cost of a dependency graph that surprises newcomers who expected "add one jar."

JUnit is used almost everywhere Java is tested — it is the assumed default in Maven Surefire, Gradle, and every major IDE — which makes backward compatibility and quiet, incremental evolution more important here than feature velocity.

## Getting Started

Maven — depend on the `junit-jupiter` aggregator and pin versions through the BOM:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.junit</groupId>
      <artifactId>junit-bom</artifactId>
      <version>5.11.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
<dependencies>
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

Gradle must be told to use the Platform explicitly — forgetting this is the single most common "my tests don't run" report:

```groovy
dependencies {
    testImplementation platform('org.junit:junit-bom:5.11.0')
    testImplementation 'org.junit.jupiter:junit-jupiter'
}
test { useJUnitPlatform() }
```

A minimal Jupiter test:

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

class CalculatorTest {

    @Test
    void addsTwoNumbers() {
        assertEquals(4, 2 + 2, "2 + 2 should be 4");
    }

    @ParameterizedTest
    @ValueSource(ints = {2, 4, 100})
    void isEven(int n) {
        assertTrue(n % 2 == 0);
    }
}
```

## Architecture / How It Works

The three-layer split is the whole design:

- **JUnit Platform** defines the `TestEngine` SPI and the `Launcher` API. IDEs (IntelliJ, Eclipse) and build tools (Surefire, Gradle) integrate against the Launcher, not against Jupiter. `junit-platform-console` provides a standalone runner, and `junit-platform-suite` lets you declare test suites with `@Suite`. Anyone can implement a `TestEngine` and be discovered via `ServiceLoader`.
- **JUnit Jupiter** is one such engine (`JupiterTestEngine`) plus the programming model most people mean by "JUnit 5": `@Test`, `@BeforeEach`, `@Nested`, `@ParameterizedTest`, `@DisplayName`, and the `Assertions` / `Assumptions` static methods.
- **JUnit Vintage** is a `TestEngine` that runs JUnit 4 (and 3) tests, so a codebase can migrate incrementally by running both engines in the same build.

The extension model is the substantive improvement over JUnit 4's runners and rules. Instead of one `@RunWith`, Jupiter has `@ExtendWith` / `@RegisterExtension` and a set of callback interfaces — `BeforeEachCallback`, `ParameterResolver`, `TestInstancePostProcessor`, `ExecutionCondition`, and others. Extensions compose: a class can use Spring, Mockito, and a temp-directory extension simultaneously. `ParameterResolver` is why Jupiter test methods can accept injected arguments (`TestInfo`, `@TempDir Path`, Spring beans) at all.

Assertions are deliberately minimal. Jupiter ships `assertEquals`, `assertThrows`, `assertAll`, and friends, but no fluent matcher DSL — the project expects you to bring AssertJ or Hamcrest for rich assertions. This keeps the core small but means "which assertion library" is a decision every project still makes.

## Production Notes

- **The dependency graph is the footgun.** `junit-jupiter` is an aggregator pulling in `-api`, `-params`, and `-engine`; the `-api` alone compiles but won't run. Mixing a Jupiter version with a mismatched Platform version produces `NoSuchMethodError` at runtime. Always import `junit-bom` and let it align every artifact — do not hard-pin individual versions.
- **Gradle silence.** Without `useJUnitPlatform()` in the `test` task, Gradle uses its legacy JUnit 4 runner and simply finds zero Jupiter tests, reporting success. There is no error — just a green build that ran nothing.
- **Surefire/Failsafe.** `maven-surefire-plugin` gained native Platform support in 2.22.0; older versions need a provider dependency. Very old Surefire configs that filter by `*Test.java` naming still apply and can hide tests.
- **Parallel execution is opt-in and config-driven.** It is disabled by default and enabled through `junit.jupiter.execution.parallel.enabled=true` plus a `junit-platform.properties` file; it stayed labeled experimental for years, and shared mutable state across parallel tests is the usual source of flakiness.
- **Test instance lifecycle.** A fresh test instance is created per method by default (`PER_METHOD`). `@TestInstance(PER_CLASS)` is required before non-static `@BeforeAll`/`@AfterAll` or non-static parameterized method sources will work — a frequent stumbling point.
- **JUnit 4 migration.** `@Rule` and `@RunWith` are not understood by Jupiter. `junit-jupiter-migrationsupport` covers a subset of rules; most rules must be rewritten as extensions. Running Vintage alongside Jupiter is the pragmatic bridge.
- **Java baseline moved.** JUnit 5 required Java 8; the JUnit 6 line raised the minimum runtime (Java 17)[^1], so upgrading past 5.x is coupled to your JDK target, not just a version bump.

## When to Use / When Not

**Use when:**
- You are writing tests for anything on the JVM — this is the default, and IDE/build-tool support assumes it.
- You want a composable extension model (Spring, Mockito, Testcontainers all plug in cleanly).
- You are migrating a JUnit 4 codebase incrementally and need both engines running side by side.

**Avoid / look elsewhere when:**
- You want data providers, groups, and suite configuration richly built in without extensions — TestNG covers more out of the box.
- You prefer a BDD/spec style — Spock (Groovy) or Kotest (Kotlin) read better for that, though both can run on the JUnit Platform.
- You only need the assertion half — JUnit's assertions are intentionally thin; pair with AssertJ regardless of framework.

## Alternatives

- testng/testng — older annotation-based JVM test framework; richer built-in suites, groups, and data providers without an extension layer.
- spockframework/spock — Groovy-based BDD/specification framework; expressive `expect`/`where` blocks, runs on the JUnit Platform.
- kotest/kotest — Kotlin-first testing with multiple spec styles and property testing; use it when the codebase is primarily Kotlin.
- assertj/assertj — not a replacement but the usual companion; add it for fluent assertions since Jupiter's are minimal.
- cucumber/cucumber-jvm — use instead when the goal is executable plain-language specifications rather than unit tests.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015 | "JUnit Lambda" crowdfunding seeds the JUnit 5 rewrite; `junit5` repo created. |
| 5.0.0 | 2017-09-10 | GA. Platform + Jupiter + Vintage split[^2]. |
| 5.4 | 2019-02 | Maturing extension model, `@Nested`/display-name improvements. |
| 5.7 | 2020-09 | Stabilization pass across the Platform. |
| 5.9 | 2022-07 | `@Suite` / `junit-platform-suite` promoted. |
| 5.11 | 2024-08 | Continued Jupiter/params refinements. |
| 6.0 | 2025 | JUnit 6 line; raised the JDK baseline; repo renamed to `junit-framework`[^1]. |
| 6.1.2 | 2026-07-12 | Latest GA at time of writing[^3]. |

## References

[^1]: JUnit project — repository `junit-team/junit-framework` (formerly `junit5`); homepage and user guide. https://junit.org
[^2]: JUnit team, "JUnit 5.0.0 = Platform + Jupiter + Vintage" release, 2017-09-10. https://github.com/junit-team/junit-framework/releases/tag/r5.0.0
[^3]: JUnit 6.1.2 release, 2026-07-12. https://github.com/junit-team/junit-framework/releases/tag/r6.1.2

## Tags

java, jvm, testing, unit-testing, test-framework, junit, junit5, jupiter, kotlin-testing, bdd, tooling
