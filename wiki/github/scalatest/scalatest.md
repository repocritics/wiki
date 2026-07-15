# scalatest/scalatest

> The most feature-complete testing framework for Scala — many testing styles, a large matcher DSL, and a companion assertion library, at the cost of weight and compile time.

[GitHub repo](https://github.com/scalatest/scalatest) ·
[Official website](https://www.scalatest.org) ·
[License: Apache-2.0](https://github.com/scalatest/scalatest/blob/main/LICENSE)

## Overview

ScalaTest is a testing toolkit for Scala and Java, created by Bill Venners (Artima) and open-sourced in the late 2000s[^1]. For most of the Scala ecosystem's history it has been the default testing framework — the one assumed by tutorials, sbt templates, and books like *Programming in Scala* (Venners is a co-author). It ships alongside **Scalactic**, a companion library of assertion, equality, and requirement utilities that can be used independently of the test framework.

Its defining trait is breadth. ScalaTest does not pick one way to write tests; it offers roughly eight interchangeable *styles* — `FunSuite`, `FlatSpec`, `FunSpec`, `WordSpec`, `FreeSpec`, `PropSpec`, `FeatureSpec`, and `RefSpec` — plus a large `Matchers` DSL (`result should be (3)`, `list should contain ("x")`), property-based testing, async variants, and integrations with JUnit, TestNG, ScalaCheck, Selenium, and mocking libraries. The intent is that a team chooses a style once and stays consistent; the effect is that ScalaTest's public surface is enormous and no two codebases use it the same way.

That flexibility is the central tension. ScalaTest is the safe, batteries-included choice, but the matcher DSL leans heavily on implicits and macros, which contributes to slow compilation, and the sheer number of styles makes the learning curve wider than lighter competitors. Since roughly 2020 a visible slice of the community has migrated to MUnit or utest for faster builds and a smaller API, while ScalaTest remains the incumbent in large and legacy codebases[^2].

## Getting Started

sbt (Scala 2.13 / 3):

```scala
// build.sbt
libraryDependencies += "org.scalatest" %% "scalatest" % "3.2.19" % Test
testFramework := TestFrameworks.ScalaTest  // sbt usually auto-detects
```

A test using the `AnyFlatSpec` style with matchers:

```scala
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers

class SetSpec extends AnyFlatSpec with Matchers {
  "An empty Set" should "have size 0" in {
    Set.empty[Int].size shouldBe 0
  }

  it should "throw NoSuchElementException when head is called" in {
    assertThrows[NoSuchElementException] {
      Set.empty[Int].head
    }
  }
}
```

The `assert` macro produces a detailed failure message by inspecting the expression — `assert(a == b)` reports both operands without a custom message. `assertThrows`, `intercept`, and `assertResult` cover the common assertion shapes.

## Architecture / How It Works

ScalaTest is built on **trait mixin composition**. The base abstraction is the `Suite` trait; every style is a trait that extends it and defines how tests are registered (`test("...")`, `"x" should "y" in {...}`, `it should ...`). You assemble a test class by mixing a style trait with optional capability traits:

- **Style trait** — e.g. `AnyFlatSpec`, `AnyFunSuite`. Determines test-declaration syntax.
- **`Matchers`** — a separate trait (`matchers.should.Matchers` or `matchers.must.Matchers`) providing the `should`/`must` DSL through implicit conversions.
- **Lifecycle traits** — `BeforeAndAfter`, `BeforeAndAfterEach`, `BeforeAndAfterAll` for setup/teardown.
- **Mixin behaviors** — `OptionValues`, `EitherValues`, `Inside`, `Inspectors`, `TableDrivenPropertyChecks`, and so on, each pulled in only when used.

This composition is the framework's core design idea: capabilities are traits you opt into, not a monolithic base class. It also means a test class's behavior is defined by its linearization order, which occasionally surprises newcomers.

**Scalactic** underlies the assertion machinery. It provides `TripleEquals` (`===`), type-checked equality (`TypeCheckedTripleEquals`), `Tolerance` for floating-point comparisons, `Requirements`, and data types like `Or`/`Every`/`Chain`. It is published as a standalone artifact and has no dependency on the test runner.

**Cross-platform and cross-version.** ScalaTest is cross-compiled to the JVM, Scala.js, and Scala Native, and published for Scala 2.11, 2.12, 2.13, and 3[^3]. Maintaining binary compatibility across this matrix is a large part of the project's build (the README's publish step runs MiMa binary-compatibility checks per module per Scala version).

**Modularization.** Since 3.2, ScalaTest is split into many artifacts (`scalatest-funsuite`, `scalatest-flatspec`, `scalatest-shouldmatchers`, etc.) so projects can depend on only the styles they use, plus the `org.scalatestplus` family for third-party integrations (ScalaCheck, JUnit, TestNG, Selenium, Mockito) that were pulled out of the core[^4].

## Production Notes

**Compile time is the main cost.** The `Matchers` DSL is implemented with implicit conversions and macros; files that use it heavily compile noticeably slower than plain assertions. Teams sensitive to build time sometimes standardize on `assert(...)` plus the `shouldBe` subset, or move to a lighter framework. Depending on the narrow `scalatest-flatspec` + `scalatest-shouldmatchers` artifacts instead of the umbrella `scalatest` dependency reduces classpath weight but not the macro-expansion cost in test sources.

**Style sprawl.** Because ScalaTest permits many styles, mixed codebases accumulate inconsistency. The maintainers' own guidance is to pick one style per team and enforce it; without that discipline, a repo ends up with `FlatSpec`, `WordSpec`, and `FunSuite` tests side by side, each reading differently.

**`should` vs `must`.** Both matcher dialects exist and are functionally identical; choosing between them is purely stylistic, but the two cannot be mixed in one class. This is a frequent source of confusion for people reading examples that use the other one.

**Property-based testing moved out.** `GeneratorDrivenPropertyChecks` and the direct ScalaCheck coupling were deprecated and relocated to the separate `scalatestplus-scalacheck` module during the 3.1/3.2 reorganization. Older code and tutorials reference the in-core API that no longer ships; upgrading requires adding the plus module and updating imports[^4].

**Async testing.** The `Async*` style variants (`AsyncFlatSpec`, etc.) let test bodies return `Future[Assertion]` so the runner does not block a thread per test. They require care: assertions must be the last expression in the future chain, and side-effecting assertions before the returned future are silently ignored.

**Upgrade friction.** The 2.x → 3.0 transition split out Scalactic and reworked packages; 3.0 → 3.2 moved matchers into `should`/`must` subpackages and externalized integrations. Each step is a mechanical but non-trivial import rewrite across a test suite.

## When to Use / When Not

**Use when:**
- You want one framework that covers unit, spec/BDD, property-based, and async testing without adding libraries.
- You value the expressive matcher DSL and rich failure diagnostics.
- You're in a large or legacy Scala codebase where ScalaTest is already the standard.
- You need Java interop or JUnit/TestNG runner integration.

**Avoid when:**
- Build/compile time is a priority — MUnit or utest compile faster with a smaller API.
- You want a minimal, uniform testing style with little to learn.
- You're in an effect-oriented (cats-effect / ZIO) codebase — effect-native frameworks integrate more cleanly.
- You want to avoid heavy implicit/macro usage in test sources.

## Alternatives

- scalameta/munit — lighter, JUnit-compatible, fast to compile and minimal API; use when you want simplicity and build speed over a large DSL.
- etorreborre/specs2 — the other comprehensive Scala framework, strong on acceptance/spec styles; use when you prefer its mutable/immutable spec model.
- com-lihaoyi/utest — very small, uniform, cross-platform testing library; use when you want one style and near-zero ceremony.
- disneystreaming/weaver-test — effect-first, integrates with cats-effect and ZIO; use in functional codebases where tests return effects.
- typelevel/scalacheck — property-based testing engine; use directly (or via a framework's plus module) when generative testing is the focus.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2009 | First 1.0 release; core styles and matchers[^1]. |
| 2.0 | 2013-12 | Major release; expanded matchers, `Assertions` reworked. |
| 3.0 | 2016-04 | Scalactic split out; async styles added[^5]. |
| 3.1 | 2019-12 | ScalaCheck integration deprecated in core, moved to scalatestplus. |
| 3.2 | 2020-06 | Modularization into per-style artifacts; matchers into `should`/`must` subpackages[^4]. |
| 3.2.19 | 2024 | Current 3.2.x line; Scala 3 and cross-platform support. |

## References

[^1]: ScalaTest — official site and user guide. https://www.scalatest.org/
[^2]: MUnit — Scala testing library emphasizing simplicity and speed, positioned as a lighter alternative. https://scalameta.org/munit/
[^3]: ScalaTest install / cross-build matrix (JVM, Scala.js, Scala Native; Scala 2.11–3). https://www.scalatest.org/install
[^4]: ScalaTest 3.2 release notes — modularization and externalized integrations. https://www.scalatest.org/release_notes/3.2.0
[^5]: ScalaTest 3.0 release notes — Scalactic separation and async testing. https://www.scalatest.org/release_notes/3.0.0

## Tags

scala, java, testing, test-framework, bdd, property-based-testing, matchers, jvm, scala-js, scala-native, assertions
