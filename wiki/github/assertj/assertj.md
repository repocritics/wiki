# assertj/assertj

> Fluent, strongly-typed assertions for Java and the JVM — `assertThat(x)` plus IDE autocompletion instead of a wall of matchers.

[GitHub repo](https://github.com/assertj/assertj) ·
[Official website](https://assertj.github.io/doc/) ·
[License: Apache-2.0](https://github.com/assertj/assertj/blob/main/LICENSE.txt)

## Overview

AssertJ is a Java assertion library for unit tests. Instead of JUnit's `assertEquals(expected, actual)` or Hamcrest's `assertThat(actual, is(...))`, you write `assertThat(actual).isEqualTo(...)` and let code completion surface every assertion valid for that type — String assertions on a `String`, Map assertions on a `Map`, Iterable assertions on a collection[^1]. The library is test-scope only: it has no runtime footprint in shipped code and no opinion about which test runner you use (JUnit 4/5, TestNG, Spock, plain `main`).

The project began as a fork of the abandoned FEST-Assert library, led by Joël Costigliola, and the current strongly-typed, fluent design dates from that lineage[^2]. Historically each type family lived in its own repository (assertj-core, assertj-guava, assertj-db, and so on); the `assertj/assertj` repo is the consolidated monorepo that now hosts core plus the satellite modules. Core is by far the most used artifact (`org.assertj:assertj-core`).

At ~2.8k stars and ~780 forks the raw popularity numbers understate its reach: AssertJ is a transitive dependency of Spring Boot's test starter and is the default assertion library in a large fraction of Java projects, so most usage arrives through frameworks rather than direct adoption. Commits land regularly (last push mid-2026), and the repo runs an explicit binary-compatibility CI gate — a signal that not breaking downstream test suites is treated as a first-class constraint.

## Getting Started

Maven:

```xml
<dependency>
  <groupId>org.assertj</groupId>
  <artifactId>assertj-core</artifactId>
  <version>3.27.3</version>
  <scope>test</scope>
</dependency>
```

Gradle:

```groovy
testImplementation("org.assertj:assertj-core:3.27.3")
```

```java
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@Test
void demo() {
    assertThat("The Lord of the Rings")
        .startsWith("The")
        .contains("Lord")
        .doesNotContain("Star Wars");

    assertThat(List.of("Frodo", "Sam", "Merry"))
        .hasSize(3)
        .contains("Frodo")
        .doesNotContain("Sauron");

    assertThatThrownBy(() -> { throw new IllegalArgumentException("boom"); })
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("boom");
}
```

The single static-import wildcard `org.assertj.core.api.Assertions.*` exposes the whole `assertThat` family; most teams add it to the IDE's auto-import favorites.

## Architecture / How It Works

The core trick is the **self-typing recursive generic** (the curiously-recurring template pattern). Every assertion class extends `AbstractAssert<SELF, ACTUAL>`, and each fluent method returns `SELF`. That is what keeps a chain strongly typed: after `.startsWith(...)` on a `StringAssert` you still see `StringAssert` methods, not a widened base type. `Assertions.assertThat(...)` is a large bank of overloads that dispatches on the static type of the argument to the right `*Assert` subclass.

Notable subsystems:

- **Soft assertions** (`SoftAssertions`, `assertSoftly`) — collect all failures in a block and report them together instead of stopping at the first. Removes the "fix one assert, rerun, find the next" loop.
- **Recursive comparison** (`usingRecursiveComparison()`) — compares two objects field-by-field, ignoring `equals()`, with per-field overrides, type comparators, and ignore rules. Heavily used to avoid writing/maintaining `equals` just for tests.
- **Exception assertions** — `assertThatThrownBy`, `assertThatExceptionOfType(...).isThrownBy(...)`, `catchThrowable`. The BDD variants (`then(...)`, `thenThrownBy`) mirror the whole API under a `BDDAssertions` entry point.
- **`InstanceOfAssertFactories`** — lets `.asInstanceOf(...)` / `.extracting(...)` narrow back to a typed assertion after a type-erasing step.
- **Condition API** (`Condition`, `allOf`, `anyOf`) — Hamcrest-matcher-style predicates for cases the fluent API doesn't cover.
- **Assertions generator** — a separate tool (`assertj-assertions-generator`) emits custom `*Assert` classes for your own domain types.

The satellite modules (Guava, DB, Swing, Joda-Time, Neo4J) reuse the same base classes to add type-specific assertions for non-JDK types. Guava and DB are actively co-released; several older modules (Swing, Joda-Time, Neo4J) are effectively legacy.

## Production Notes

- **This is a test dependency.** Keep it at `test` scope. It has no place on the runtime classpath and adds nothing to your artifact.
- **Version alignment matters across modules.** assertj-core and the satellite modules are versioned together in the monorepo; mixing an old satellite jar with a newer core (common when Spring Boot's dependency management pins one) can produce `NoSuchMethodError` at test time. Let the BOM/platform manage the version rather than pinning by hand.
- **Java baseline is a real upgrade gate.** The 3.x line targets Java 8+. The 4.x line raises the floor to Java 17 and removes long-deprecated APIs[^3]. A test suite that compiled clean on 3.x can fail to compile on 4.x; treat the major bump as a code change, not a version bump.
- **Recursive comparison is powerful and a footgun.** On object graphs with cycles it needs explicit cycle handling, and on large/deep graphs it is slow and can produce enormous diff messages. Scope it with `.ignoringFields(...)` / `.comparingOnlyFields(...)` rather than comparing whole aggregates.
- **`extracting(String...)` uses reflection on field/property names** — it is not refactor-safe. String-based property paths silently rot when a field is renamed; the lambda-based `extracting(Type::getter)` overloads are type-checked and preferred.
- **Overload ambiguity.** Projects that also use Truth or a custom `assertThat` can hit static-import collisions; qualify or pick one per test module.
- **The value is the failure message.** AssertJ's differentiator over `assertEquals` is the human-readable diff on failure (element-level list diffs, field-level object diffs). If a custom assertion throws a bare `AssertionError` without describing the mismatch, you have lost the main reason to use the library — prefer `failWithMessage`/`Descriptable.as(...)` when extending it.

## When to Use / When Not

**Use when:**
- You write JVM unit or integration tests and want readable, discoverable assertions with good failure output.
- You need collection, exception, or deep-object assertions that would be verbose in plain JUnit or Hamcrest.
- You want soft assertions to see every failure in one run.
- You're on Spring Boot / a JVM test stack where it's already present.

**Avoid / reconsider when:**
- You're pinned to Java 8–16 and want the latest line — you must stay on 3.x (still maintained) rather than 4.x.
- Your team prefers Kotlin-idiomatic assertions — Kotest or Strikt read more naturally in Kotlin than AssertJ's Java-first fluent chains.
- You only need a handful of trivial equality checks — JUnit 5's built-in assertions add no dependency.
- You're testing asynchronous conditions — pair with Awaitility; AssertJ alone does not poll.

## Alternatives

- hamcrest/JavaHamcrest — older matcher-composition model; more extensible for custom matchers but far more verbose and weaker autocompletion. Use when you need composable matchers reused across libraries.
- junit-team/junit5 — built-in `Assertions` cover basic equality/exception cases with zero extra dependency. Use when you want nothing beyond the test runner.
- google/truth — Google's fluent assertion library with a similar `assertThat` style. Use when you prefer its stricter, smaller, more opinionated surface and failure-message philosophy.
- kotest/kotest — Kotlin-native matchers and property testing. Use instead of AssertJ when the codebase is primarily Kotlin.
- awaitility/awaitility — not a replacement but a complement: use alongside AssertJ to assert conditions that become true asynchronously.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | ~2013 | Forked from the abandoned FEST-Assert library; strongly-typed fluent design established[^2]. |
| 2.x | 2015 | Java 7 baseline line. |
| 3.0 | 2016 | Java 8 baseline; lambda-friendly APIs (exception assertions, `extracting` with method refs)[^1]. |
| 3.x | 2016–2025 | Long-lived line: soft assertions, recursive comparison, `InstanceOfAssertFactories`, steady module releases. |
| 4.0 | 2025 | Java 17 baseline; removal of deprecated APIs; consolidated monorepo era[^3]. |

Exact per-minor release dates are not restated here; see the GitHub releases page for the authoritative changelog[^4].

## References

[^1]: AssertJ Core documentation and assertions guide. https://assertj.github.io/doc/
[^2]: AssertJ project background / FEST-Assert fork lineage (repo README and issue history). https://github.com/assertj/assertj
[^3]: AssertJ release notes (4.0 Java 17 baseline and deprecated-API removal). https://github.com/assertj/assertj/releases
[^4]: AssertJ releases (authoritative version/date changelog). https://github.com/assertj/assertj/releases

## Tags

java, jvm, testing, assertions, fluent-api, unit-testing, test-framework, junit, hamcrest-alternative, apache-2.0
