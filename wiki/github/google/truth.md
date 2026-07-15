# google/truth

> A fluent assertion library for Java and Android, built and used internally by Google, optimized for readable failure messages over API breadth.

[GitHub repo](https://github.com/google/truth) ·
[Official website](https://truth.dev/) ·
[License: Apache-2.0](https://github.com/google/truth/blob/master/LICENSE)

## Overview

Truth is a test-assertion library owned and maintained by the Guava team at Google, and used across the majority of Google's own Java test suite[^1]. Its API is the `assertThat(actual).isEqualTo(expected)` fluent form: you wrap the value under test in a `Subject`, then call assertion methods that read close to English. It is not a test runner — it throws `AssertionError` and slots underneath JUnit, TestNG, or anything else.

The design philosophy is deliberately narrow. Where AssertJ competes on the sheer number of built-in assertion methods, Truth competes on the quality of the failure message when an assertion fails, and on a small, consistent, hard-to-misuse API surface[^2]. Truth models failures as structured `Fact` key-value pairs rather than free-form strings, so a failing `containsExactly` prints the missing and unexpected elements in a stable, diff-like layout instead of a wall of `toString()`.

The defining tension is scope. Truth gives you fewer assertion methods than AssertJ and no fluent multi-assertion chaining on a single subject; in exchange you get terser failure output and a codebase that resists the "which of forty overloads do I call" problem. Teams that want an exhaustive built-in method for every shape of data usually reach for AssertJ instead; teams that value message readability and a curated surface pick Truth.

## Getting Started

Maven:

```xml
<dependency>
  <groupId>com.google.truth</groupId>
  <artifactId>truth</artifactId>
  <version>1.4.4</version>
  <scope>test</scope>
</dependency>
```

Gradle: `testImplementation("com.google.truth:truth:1.4.4")`

```java
import static com.google.common.truth.Truth.assertThat;
import static com.google.common.truth.Truth.assertWithMessage;

@Test
public void demo() {
  assertThat("apple").startsWith("app");
  assertThat(List.of(1, 2, 3)).containsExactly(3, 2, 1);   // order-independent
  assertThat(Map.of("k", "v")).containsEntry("k", "v");

  // custom message prefix
  assertWithMessage("user %s should be active", id)
      .that(user.isActive())
      .isTrue();
}
```

A failing `containsExactly` prints the specific missing/unexpected elements rather than two full `toString()` dumps — the core value proposition.

## Architecture / How It Works

The whole library is organized around one abstraction: `Subject`. `assertThat(x)` is a set of overloaded static factories on the `Truth` class that dispatch to a type-specific subject — `StringSubject`, `IterableSubject`, `MapSubject`, `IntegerSubject`, `ThrowableSubject`, and so on. Each subject exposes only the assertions that make sense for its type, so autocomplete on a `String` never offers you `containsExactly`.

Custom subjects are the extension mechanism. You subclass `Subject`, expose a `Subject.Factory`, and call `assertAbout(myFactory).that(value).myAssertion()`. This is how the proto and Guava extensions are built, and how domain code adds first-class assertions for its own types[^3].

Failure reporting goes through `Subject.failWithActual` / `failWithoutActual`, which emit lists of `Fact` objects (a `key: value` pair). The framework, not the assertion author, decides final formatting — alignment, truncation of large collections, and diff presentation are centralized. This is why Truth's messages are uniform across assertion types.

Three JUnit integration points sit on top of the core:

- **`Expect`** — a JUnit `Rule`/extension that collects failures and reports them all at the end of the test instead of stopping at the first (soft assertions).
- **`TruthJUnit.assume()`** — converts a failed assertion into a JUnit *assumption* failure, so the test is skipped rather than failed.
- **`assertWithMessage(...)`** and **`assertAbout(...)`** — the two entry points that wrap the standard `assertThat`.

Truth depends on Guava, which is both a feature (native subjects for `Multimap`, `Optional`, `Table`, etc.) and a transitive-dependency cost.

## Production Notes

**Truth8 was folded into the main API.** For years, assertions over `java.util.Optional`, `Stream`, `OptionalInt`, and other Java 8 types lived on a separate `Truth8` class. Starting in the 1.4.0 line those methods were merged onto the standard `Truth`/`assertThat` entry points and `Truth8` was deprecated[^4]. Migrations that upgrade across this boundary need to drop `Truth8.assertThat` imports and re-point them at `Truth.assertThat`; leaving both can cause ambiguous static imports.

**`containsExactly` argument order and varargs.** `containsExactly(1, 2, 3)` checks contents ignoring order; to also assert order you chain `.inOrder()`. Passing a single `Iterable` to a varargs `containsExactly` is a classic footgun — it asserts the collection contains *one element which is that iterable*. Use `containsExactlyElementsIn(collection)` for the collection-vs-collection case.

**Arrays.** Java arrays don't have value `equals`, so `assertThat(someArray)` returns a primitive/object array subject with methods like `asList()` and `isEqualTo` that compare by content; don't assume raw `.isEqualTo` on two arrays behaves like reference equality.

**No single-statement chaining.** Unlike AssertJ, you cannot chain multiple distinct assertions on one subject (`assertThat(s).startsWith("a").endsWith("z")` is not the pattern). Each check is its own statement, or you use `Expect` for soft grouping. Teams migrating from AssertJ notice this immediately.

**Method breadth.** Truth intentionally ships fewer built-in assertions. Complex structural checks (recursive field comparison, extracting properties, custom comparators over collections) are far more turnkey in AssertJ. In Truth you either write a custom `Subject` or assert the derived value directly.

**Android.** Truth runs on Android without the reflection/JDK-only pitfalls that historically affected some assertion libraries, which is a large part of why Google standardized on it internally.

## When to Use / When Not

**Use when:**
- You value uniform, readable failure messages over raw method count.
- You want a small, consistent, hard-to-misuse assertion surface.
- You already use Guava and want native subjects for its collection types.
- You test on both the JVM and Android and want one assertion library.
- You want to add domain-specific assertions cleanly via custom `Subject`s.

**Avoid when:**
- You want the largest possible set of built-in assertions out of the box — AssertJ has more.
- You rely on fluent multi-assertion chaining on a single object.
- You need recursive/field-by-field object comparison as a first-class built-in.
- You want zero extra dependencies — Truth pulls in Guava.

## Alternatives

- assertj/assertj — richer fluent API with far more built-in assertions and recursive comparison; use it when you want breadth and single-subject chaining over message minimalism.
- hamcrest/JavaHamcrest — matcher-composition model; use it when an API (older JUnit `assertThat`, Mockito argument matchers) expects `Matcher` objects.
- junit-team/junit5 — the built-in `Assertions` class covers `assertEquals`/`assertThrows`; use it when you want no third-party assertion dependency at all.
- kotest/kotest — idiomatic assertions for Kotlin codebases; use it instead when the project is Kotlin-first.
- google/guava — same team, but a general utility library, not an assertion framework; not a substitute.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2011-06 | Repository created; early internal Google assertion library[^1]. |
| 0.x series | 2011–2019 | Long pre-1.0 run; API stabilized as `Subject`-based fluent assertions. |
| 1.0 | 2019 | First stable release; API-compatibility commitment begins[^5]. |
| 1.4.0 | 2024 | `Truth8` methods merged into main `Truth` API; `Truth8` deprecated[^4]. |

## References

[^1]: Truth README — "owned and maintained by the Guava team … used in the majority of the tests in Google's own codebase." https://github.com/google/truth
[^2]: Truth documentation, "Comparison to AssertJ and Hamcrest." https://truth.dev/comparison
[^3]: Truth documentation, "Writing your own custom subject / Extension." https://truth.dev/extension
[^4]: Truth release notes / `Truth8` deprecation (1.4.0 line, 2024). https://github.com/google/truth/releases
[^5]: Truth release history. https://github.com/google/truth/releases

## Tags

java, android, testing, assertion-library, unit-testing, junit, test-framework, google, guava, fluent-assertions
