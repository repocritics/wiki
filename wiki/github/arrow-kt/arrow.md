# arrow-kt/arrow

> Typed functional programming for Kotlin — Option/Either, typed errors, structured-concurrency helpers, and optics, built to feel like idiomatic Kotlin rather than ported Haskell.

[GitHub repo](https://github.com/arrow-kt/arrow) ·
[Official website](https://arrow-kt.io) ·
License: Apache-2.0

## Overview

Arrow is a companion library for Kotlin that supplies the data types and control-flow constructs functional programmers expect: `Option`, `Either`, `Ior`, `NonEmptyList`, a typed-error DSL, resource-safe concurrency, and profunctor optics. It is not a framework — it adds no runtime, no DI container, and no application scaffolding. It aims to be a *lingua franca* of FP abstractions that other Kotlin libraries can share[^1].

The defining fact about Arrow is that it is a library in its second identity. Arrow was formed in 2017 by merging two earlier projects, KΛTEGORY and funKTionale[^2], and for its first several years it tried to reproduce the Scala/cats style of higher-kinded typeclasses on a language that has no higher-kinded types. That required an emulation layer (`Kind<F, A>` witnesses, `@higherkind` codegen, `arrow-meta` compiler plugins) that was verbose, slow to compile, and alien to most Kotlin developers. Around the 1.0 release (2021) the maintainers deliberately abandoned that direction[^3]: typeclasses and HKT emulation were dropped, and the library was rebuilt around concrete data types and Kotlin's own suspension mechanism. Computation blocks that once needed monad typeclasses became coroutine-backed DSLs (`either { }`, `nullable { }`, `resource { }`).

The practical consequence is that Arrow's history actively works against newcomers: a large fraction of blog posts, StackOverflow answers, and tutorials describe the pre-1.0 typeclass API, which no longer exists. Reading current documentation and ignoring anything about `Kind`, typeclasses, or `arrow-meta` is the single most useful onboarding rule.

## Getting Started

Arrow ships as separate Gradle artifacts under `io.arrow-kt`. Add only the modules you use:

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.arrow-kt:arrow-core:2.1.2")
    implementation("io.arrow-kt:arrow-fx-coroutines:2.1.2")
}
```

A minimal typed-error example using the `Raise` DSL:

```kotlin
import arrow.core.Either
import arrow.core.raise.either
import arrow.core.raise.ensure

data class User(val name: String)

fun validate(name: String): Either<String, User> = either {
    ensure(name.isNotBlank()) { "name must not be blank" }
    ensure(name.length <= 50) { "name too long" }
    User(name)
}

fun main() {
    println(validate(""))        // Either.Left(name must not be blank)
    println(validate("Tom"))     // Either.Right(User(name=Tom))
}
```

`either { }` runs the block; the first `raise(...)` (directly or via `ensure`/`bind`) short-circuits to a `Left`, and falling off the end produces a `Right`.

## Architecture / How It Works

Arrow is a set of independent modules, not a monolith:

- **arrow-core** — the concrete types (`Option`, `Either`, `Ior`, `NonEmptyList`, `EitherNel`) and the `Raise<E>` DSL. `Raise<E>` is the load-bearing abstraction of modern Arrow: an interface with a `raise(e: E)` capability, consumed as a context receiver. `either { }`, `result { }`, `nullable { }`, and `option { }` are thin runners over it. `zipOrAccumulate` and `mapOrAccumulate` provide error *accumulation* (collect all failures) rather than fail-fast, which is Arrow's answer to validation.
- **arrow-fx-coroutines** — structured-concurrency helpers layered on `kotlinx.coroutines`, not a replacement for it. `parZip` / `parMap` run tasks in parallel with cancellation propagation; `Resource` and the `resource { }` DSL give bracket-style acquire/release guarantees that survive cancellation and exceptions.
- **arrow-resilience** — `Schedule` (composable retry/repeat policies), `CircuitBreaker`, and `Saga` (compensating transactions).
- **arrow-optics** — `Lens`, `Prism`, `Optional`, `Traversal`, `Iso` for immutable deep updates. The `@optics` annotation drives a KSP compiler plugin that generates boilerplate optics for your data classes.

The `Raise` design is what makes Arrow feel native. Because `raise` is a plain function call inside a normal Kotlin block (not a monadic `flatMap` chain), control flow reads like imperative code with early return, while remaining pure at the type level. Under the hood the short-circuit is implemented with a control-flow exception that never escapes the DSL boundary — cheap, but it is an exception, which matters for a few edge cases (see Production Notes).

Optics work by composition: a `Lens<Company, Street>` is built by composing `Lens<Company, Address>` with `Lens<Address, Street>`, and `.modify { }` produces a new immutable copy with one field changed at arbitrary depth.

## Production Notes

**The pre-1.0 knowledge gap is the biggest real cost.** Because the library changed shape entirely, older material is not just outdated — it references APIs that were deleted. Budget for the team learning the current model rather than transferring cats/Scalaz intuition.

**`Raise` uses exceptions for control flow.** The short-circuit in `either { }` / `Raise` is a `ControlThrowable`. Two consequences: (1) a `try { ... } catch (e: Throwable) { ... }` *inside* a `Raise` block can accidentally swallow the short-circuit signal — catch specific exceptions, not `Throwable`; (2) crossing the DSL boundary inside a lambda passed to code that catches broadly can misbehave. Arrow provides `Raise`-aware combinators to avoid this, but it is a genuine footgun.

**Context receivers and the Kotlin roadmap.** Arrow leaned into `context(Raise<E>)` receivers, a Kotlin language feature that has been experimental and reworked (context receivers are being replaced by "context parameters"). Code written against one incarnation may need adjustment as the language feature stabilizes. Function-argument and extension-receiver styles (`Raise<E>.()` ) remain the portable fallback.

**Module granularity and version alignment.** Every module is versioned in lockstep; mixing, e.g., `arrow-core:2.1.2` with an older `arrow-optics` will produce binary incompatibilities. Use a single version constant or the Arrow BOM (`arrow-stack`) to keep them aligned.

**Optics codegen.** `@optics` requires wiring the KSP plugin; without it the generated companion optics simply don't exist and you get unresolved references. On large data models the generated code inflates compile time.

**Multiplatform.** Arrow is a Kotlin Multiplatform library (JVM, JS, Native, Wasm targets for most modules). Target availability can lag per module and per release, so verify the specific artifact publishes for your target before committing.

## When to Use / When Not

**Use when:**
- You want typed, exhaustive error handling (`Either` / `Raise`) instead of exceptions, with the option to accumulate validation errors.
- You already use `kotlinx.coroutines` and want resource-safe parallelism and retry/circuit-breaker policies.
- You need deep immutable updates and want optics instead of nested `copy()` pyramids.
- You want small, à-la-carte modules rather than an all-or-nothing framework.

**Avoid when:**
- The team has no appetite for FP idioms — plain Kotlin `Result`, sealed classes, and `require`/`check` cover many cases without a dependency.
- You need long-term API stability above all: Arrow has already made one wholesale identity change and rides experimental Kotlin language features.
- Your codebase is small enough that the conceptual overhead (Raise, optics, `bind`) outweighs the payoff.

## Alternatives

- kotlin/kotlinx.coroutines — for concurrency alone, Arrow-fx sits on top of it; use it directly when you don't need bracket/Resource/parZip sugar.
- michaelbull/kotlin-result — a small, focused `Result<V, E>` type; use it when you want typed errors without the rest of Arrow.
- Kotlin stdlib `Result` + sealed classes — use when built-in error modelling is enough and you want zero dependencies.
- typelevel/cats (Scala) — the conceptual ancestor; relevant only if you're on Scala, not Kotlin.
- optics via kotlinx.serialization + manual `copy` — use when your immutable updates are shallow and don't justify a lens library.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017–2020 | Formed by merging KΛTEGORY + funKTionale; typeclass / HKT-emulation era (`Kind<F,A>`, arrow-meta)[^2]. |
| 1.0.0 | 2021-10 | Pivot away from HKT/typeclasses toward concrete types and coroutine-backed computation blocks[^3]. |
| 1.2.0 | 2023 | `Raise<E>` DSL matured as the central typed-error mechanism. |
| 2.0.x | 2024–2025 | Removal of deprecated 1.x APIs; `Raise`/context-based design consolidated. |
| 2.1.2 | 2025-05 | Latest release at time of writing[^4]. |

## References

[^1]: Arrow README and project site — "a library for Typed Functional Programming in Kotlin." https://arrow-kt.io
[^2]: Arrow origin as a merge of KΛTEGORY and funKTionale (2017). https://arrow-kt.io/docs/quickstart/
[^3]: Arrow 1.0 release notes / migration — deprecation of typeclasses and higher-kinded-type encoding. https://arrow-kt.io/learn/quickstart/
[^4]: Maven Central, `io.arrow-kt:arrow-core` — latest published version 2.1.2 (2025-05). https://central.sonatype.com/artifact/io.arrow-kt/arrow-core

## Tags

kotlin, functional-programming, jvm, typed-errors, either-monad, optics, coroutines, kotlin-multiplatform, library, resilience
