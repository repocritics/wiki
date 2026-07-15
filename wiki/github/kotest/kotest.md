# kotest/kotest

> A Kotlin-native test framework bundling multiple spec styles, a standalone matcher library, and property-based testing — three tools that ship together but adopt separately.

[GitHub repo](https://github.com/kotest/kotest) ·
[Official website](https://kotest.io) ·
[License: Apache-2.0](https://github.com/kotest/kotest/blob/master/LICENSE)

## Overview

Kotest is a testing toolkit for Kotlin with multiplatform support (JVM, JS, Native)[^1]. It began life as **KotlinTest** and was renamed to Kotest with the 4.0 release in 2020[^2], a rename that came with a full package migration (`io.kotlintest` → `io.kotest`). As of 2026 it is the most-adopted Kotlin-first alternative to plain JUnit, at ~4,800 stars and pushed to daily — the commit cadence and issue turnaround indicate an actively maintained project, not a coasting one.

The project is really three loosely-coupled products under one name: (1) a **test framework** with ten-plus "spec styles" (StringSpec, FunSpec, DescribeSpec, BehaviorSpec, ShouldSpec, WordSpec, FreeSpec, and more) that let you pick BDD, xUnit, or Gherkin-flavored layouts; (2) an **assertions/matcher library** (`shouldBe`, `shouldContain`, hundreds of typed matchers) usable with any runner including vanilla JUnit; and (3) a **property-testing engine** (`kotest-property`) with generators and shrinking. The defining tension is that this modularity is real but under-advertised: many teams pull in the whole framework when they only wanted the matchers, and inherit the framework's lifecycle model and isolation semantics as a result.

Kotest leans hard into Kotlin idioms — test bodies are coroutine-capable (`suspend` works everywhere), tests are declared as data inside a spec class rather than as annotated methods, and DSLs replace annotations. That is its appeal over JUnit for Kotlin shops and also its cost: it is not a drop-in for existing JUnit test suites and its run model differs from what Java tooling assumes.

## Getting Started

On the JVM, Kotest runs on the JUnit Platform, so Gradle must be told to use it:

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("io.kotest:kotest-runner-junit5:5.9.1")
    testImplementation("io.kotest:kotest-assertions-core:5.9.1")
    testImplementation("io.kotest:kotest-property:5.9.1")
}

tasks.test {
    useJUnitPlatform()   // required — Kotest does not run under JUnit 4
}
```

```kotlin
import io.kotest.core.spec.style.StringSpec
import io.kotest.matchers.shouldBe
import io.kotest.property.checkAll

class MathTest : StringSpec({
    "addition is commutative" {
        (2 + 3) shouldBe (3 + 2)
    }

    "reversing twice is identity" {
        checkAll<String> { s ->
            s.reversed().reversed() shouldBe s
        }
    }
})
```

## Architecture / How It Works

Kotest publishes as a fan of small artifacts rather than one jar. The three that matter most:

- **`kotest-framework-engine`** — the multiplatform test engine that discovers specs, builds the test tree, and executes it. This is the piece that runs on Kotlin/JS and Kotlin/Native, where there is no JUnit.
- **`kotest-runner-junit5`** — a JVM-only adapter that plugs the engine into the JUnit Platform, so IDEs, Gradle, and CI that speak JUnit can drive Kotest. On the JVM this is what you actually depend on.
- **`kotest-assertions-*`** and **`kotest-property`** — the matcher and property libraries, deliberately independent of the engine so they can be used from JUnit or any other runner.

A **spec** is a class; its constructor body registers tests as lambdas. The engine walks that tree and runs it. Because tests are lambdas rather than methods, a single spec class holds shared mutable state unless you tell it otherwise — which is where **isolation modes** come in. The default is `IsolationMode.SingleInstance`: one instance of the spec class for all its tests, so a `var` declared in the spec is shared across every test. `InstancePerTest` and `InstancePerLeaf` re-instantiate the spec for finer isolation at the cost of re-running enclosing blocks[^3]. This model is the source of most Kotest surprises.

Lifecycle hooks (`beforeTest`, `afterTest`, `beforeSpec`, `afterSpec`) and cross-cutting behavior are provided through an **extension** system. Global configuration lives in a class extending `AbstractProjectConfig`, auto-discovered at startup. First-party extension modules exist for Spring, Ktor, Testcontainers, Arrow, and others, published as separate artifacts.

Property testing is built around `Arb` (random, shrinking generators) and `Exhaustive` (finite enumerations). On failure the engine shrinks the input toward a minimal reproducing case and reports a seed so the failure can be replayed.

## Production Notes

**Isolation mode is the top footgun.** Because the default `SingleInstance` shares one spec object across all tests, mutable fields leak state between tests and test order can matter. Teams migrating from JUnit (method-per-test, fresh instance each time) hit flaky tests until they either switch to `InstancePerLeaf` or stop putting mutable state in the spec body. There is no free lunch: per-leaf isolation re-executes parent blocks, which is slower and re-runs setup code.

**JUnit Platform is mandatory on the JVM.** Forgetting `useJUnitPlatform()` in Gradle produces the classic "no tests found" — the tests compile and are silently not run. This bites nearly every first-time setup.

**IDE integration needs the plugin.** The IntelliJ Kotest plugin (marketplace id 14080) provides gutter run icons and per-test running. Without it you can still run specs as classes, but the ergonomic "run this one test" experience depends on the plugin, and plugin compatibility occasionally lags new IntelliJ releases.

**Migration cost is real.** The KotlinTest → Kotest 4.0 rename changed every package and artifact coordinate; the 4→5 jump reorganized assertion artifacts (`kotest-assertions-core` split-out) and removed long-deprecated APIs. Neither is a version-bump-and-go. Pin exact versions across all Kotest artifacts — mixing engine and assertions versions produces obscure linkage errors.

**Multiplatform caveats.** The framework runs on JS and Native, but the JVM is the best-supported target by a wide margin. Native test discovery and reporting are less mature, and some assertion/extension modules are JVM-only. Verify per-target support before committing a multiplatform project to Kotest for its non-JVM tests.

**Property-test defaults.** `checkAll` runs a default number of iterations (1000 on the JVM) per invocation; heavyweight property tests can dominate suite runtime. Tune iteration counts and use `Exhaustive` for small finite domains instead of random sampling.

## When to Use / When Not

**Use when:**
- You want a Kotlin-idiomatic test experience (coroutine-native tests, DSL specs, no annotations).
- You want property-based testing and matchers in the same toolkit as your test runner.
- You value choice of spec style (BDD/Gherkin/xUnit) across a team with mixed preferences.
- You're testing coroutine-heavy code and want first-class `suspend` support.

**Avoid when:**
- You have a large existing JUnit suite and no appetite for a parallel test model.
- Your project is Java-first or mixed — JUnit 5 interops more cleanly with Java code and tooling.
- You only need assertions: pull in a standalone matcher library instead of the whole framework.
- You need the most mature non-JVM (Native) test tooling — the JVM story is far ahead of the others.

## Alternatives

- junit-team/junit5 — the platform standard; use when you want maximum Java interop and the broadest tooling support.
- willowtreeapps/assertk — fluent Kotlin assertions only; use when you want Kotest-style matchers without adopting its framework.
- robfletcher/strikt — assertion library with an expression-tree model; use when you want rich, chainable assertions on JUnit.
- mockk/mockk — Kotlin mocking library; complementary rather than competing, commonly paired with either Kotest or JUnit.
- spekframework/spek — earlier Kotlin BDD spec framework; use only if you specifically want Spek's style, as its momentum has faded relative to Kotest.

## History

| Version | Date | Notes |
|---------|------|-------|
| KotlinTest 1.x | 2016 | Original project, Kotlin spec-style testing on JUnit. |
| 4.0 | 2020 | Renamed KotlinTest → Kotest; package migration `io.kotlintest` → `io.kotest`[^2]. |
| 5.0 | 2021 | Assertion artifacts reorganized; deprecated APIs removed[^4]. |
| 5.9.x | 2024 | Current stable line at time of writing[^4]. |
| 6.0 | in progress | Next major, tracked via milestone/pre-releases; not yet stable[^4]. |

## References

[^1]: Kotest documentation and quick start. https://kotest.io/docs/quickstart/
[^2]: Kotest 4.0 introduced the rename from KotlinTest and the `io.kotest` package migration. https://kotest.io/docs/framework/migration/framework-3-to-4.html
[^3]: Kotest isolation modes reference. https://kotest.io/docs/framework/isolation-mode.html
[^4]: Kotest releases and changelog. https://github.com/kotest/kotest/releases

## Tags

kotlin, testing, test-framework, property-testing, assertions, matchers, kotlin-multiplatform, junit-platform, bdd, jvm, coroutines
