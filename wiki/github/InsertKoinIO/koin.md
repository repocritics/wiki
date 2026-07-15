# InsertKoinIO/koin

> A runtime dependency-injection framework for Kotlin and Kotlin Multiplatform, built on a DSL instead of code generation.

[GitHub repo](https://github.com/InsertKoinIO/koin) ·
[Official website](https://insert-koin.io) ·
[License: Apache-2.0](https://github.com/InsertKoinIO/koin/blob/main/LICENSE.txt)

## Overview

Koin is a dependency-injection framework for Kotlin created by Arnaud Giuliani and now maintained by Kotzilla alongside open-source contributors[^1]. Its defining choice is that the dependency graph is expressed as a Kotlin DSL (`module { single { ... } }`) and resolved at runtime, rather than generated at compile time by an annotation processor. This is what makes it "lightweight": there is no `kapt`/KSP round-trip in the classic setup, no generated factory classes, and the whole API is ordinary Kotlin functions and lambdas.

That runtime model is the framework's central tradeoff. For most of Koin's history a missing or mis-typed binding did not surface until the code path that resolved it actually ran — a `NoDefinitionFoundException` at runtime instead of a red build. Dagger/Hilt users accept boilerplate and slower builds in exchange for a compiler that proves the graph is complete; Koin users historically accepted the opposite bargain. Recent versions narrow this gap: the Koin Compiler Plugin (Koin Annotations, KSP-based) verifies definitions at compile time and generates the module code from `@Single`/`@Factory`/`@Module` annotations, giving a compile-safe path for teams that want it[^2].

Koin's second reason for existing is Kotlin Multiplatform. The core is pure Kotlin with no JVM/Android assumptions, so the same graph runs on Android, iOS, JVM, JS, and native targets, with thin integration modules (`koin-android`, `koin-androidx-compose`, `koin-compose`, `koin-ktor`) layering platform ergonomics on top[^3].

## Getting Started

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.insert-koin:koin-core:4.0.0")
    // Android: implementation("io.insert-koin:koin-android:4.0.0")
}
```

```kotlin
import org.koin.core.context.startKoin
import org.koin.dsl.module
import org.koin.core.component.KoinComponent
import org.koin.core.component.inject

interface UserRepository
class UserRepositoryImpl : UserRepository
class GetUser(private val repo: UserRepository)

val appModule = module {
    single<UserRepository> { UserRepositoryImpl() }  // one shared instance
    factory { GetUser(get()) }                        // new instance each get()
}

fun main() {
    startKoin { modules(appModule) }
    val useCase: GetUser by ServiceHolder.inject()
}

object ServiceHolder : KoinComponent
```

On Android you typically call `startKoin` in `Application.onCreate`, then resolve with `by inject()` in components or `by viewModel()` / `koinViewModel()` in Compose.

## Architecture / How It Works

A running Koin is a registry (`Koin` instance) holding *definitions* keyed by type (and an optional qualifier). The DSL is a builder over that registry:

- `single { }` — one instance per Koin instance, created lazily on first resolution (or eagerly with `createdAtStart = true`).
- `factory { }` — a fresh instance on every resolution.
- `scoped { }` — an instance tied to a `Scope` lifetime (e.g. an Android Activity/Fragment, a user session).
- `get()` inside a definition resolves a constructor argument by type; `parametersOf(...)` injects runtime values.

Resolution is a map lookup by `KClass`, not reflection over constructors — Koin does not scan or instantiate your classes reflectively; you write the `{ Foo(get()) }` lambda that does the wiring. This keeps it fast and R8/ProGuard-friendly relative to reflection-based containers, at the cost of writing the wiring by hand.

The framework leans on a **global default context** (`GlobalContext`): `startKoin`/`inject`/`KoinComponent` all reach for a single process-wide instance. That is convenient for apps but a footgun for library authors and for anything wanting multiple independent graphs — the isolated alternative is `koinApplication { }`, which returns a `Koin` you hold yourself instead of registering globally.

The Koin Annotations path is a separate front end: KSP processes `@Module`/`@Single`/`@Factory`, generates the equivalent DSL, and (in verify mode) fails the build on a missing binding — turning the runtime container into a compile-checked one without changing the runtime engine underneath[^2].

## Production Notes

- **Missing-binding errors are runtime by default.** Without the annotation/verify path, a broken graph throws at resolution. Guard it in tests: `koin-test`'s `checkModules()` / `verify()` walks every definition and fails if a dependency can't be constructed[^4]. Run it in CI — it is the single highest-value Koin test you can write.
- **Global state and test isolation.** Because `startKoin` mutates a global, tests that each `startKoin` will throw "A Koin Application has already been started" unless you `stopKoin()` between them (or use `KoinTest` rules). Parallel test execution against the global context is unsafe; prefer `koinApplication { }` for isolatable graphs.
- **KMP / iOS.** Start Koin once from common or platform code; on iOS the Kotlin framework holds the container, so re-initializing (e.g. across SwiftUI previews or test runs) needs an explicit `stopKoin()` first. There is no per-thread magic — a single container is shared.
- **Performance.** Resolution is a hash lookup and is cheap in practice; the historical "Koin is slow" criticism was largely about early reflection use and has been reduced across the 3.x line. It will not match Dagger's generated, zero-lookup code, but for typical app graphs the difference is not measurable at startup. Prefer `factory` deliberately — a `factory` in a hot loop allocates every call.
- **Android ViewModel wiring** moved into dedicated artifacts over the 3.x→4.x transition; `viewModel { }` and `koinViewModel()` live in the ViewModel/Compose modules, and imports shift between versions — a common source of "unresolved reference" churn on upgrade.

## When to Use / When Not

**Use when:**
- You want DI on Kotlin Multiplatform with one graph across Android/iOS/JVM.
- You value a small, readable DSL and fast iteration over compile-time proofs.
- Your team will add `checkModules()`/verify (or Koin Annotations) to recover safety.
- You're building on Ktor, Compose Multiplatform, or an Android app and want thin, idiomatic integration.

**Avoid when:**
- You require the graph proven complete by the compiler and won't adopt the annotation path — Dagger/Hilt or kotlin-inject fit better.
- You need zero-overhead, generated injection for a very large Android app where build-time verification is a hard requirement.
- You want to avoid global mutable state entirely and the team won't standardize on `koinApplication { }`.

## Alternatives

- google/dagger — use when you want compile-time-verified, zero-runtime-lookup DI (Hilt on Android) and will accept the boilerplate and build cost.
- evant/kotlin-inject — use when you want compile-time DI code generation for Kotlin Multiplatform without Dagger's JVM/annotation-processing weight.
- Kodein-Framework/Kodein-DI — use when you want a Koin-like runtime DSL container with a different API and long-standing multiplatform support.
- square/anvil — use when you're already on Dagger and want a Kotlin compiler plugin to cut its module/boilerplate ceremony (Android/JVM).
- Plain constructor injection — use when the graph is small enough that any container adds more ceremony than it removes.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017 | First public commits; DSL-based DI for Kotlin[^1]. |
| 1.0 | 2018 | First stable release. |
| 2.0 | 2019 | API rework, scopes overhaul, performance work. |
| 3.0 | 2021 | Kotlin Multiplatform core; pure-Kotlin engine[^3]. |
| 3.5 | 2023 | Stabilized line later designated an LTS baseline[^5]. |
| 4.0 | 2024 | Compose Multiplatform focus, KMP/ViewModel module restructuring. |
| 4.x | 2025–2026 | Compile-safe path via Koin Compiler Plugin / Annotations[^2]. |

## References

[^1]: Koin — repository and project background, maintained by Kotzilla and contributors. https://github.com/InsertKoinIO/koin
[^2]: Koin Compiler Plugin / Koin Annotations (KSP, compile-time verification). https://github.com/InsertKoinIO/koin-compiler-plugin
[^3]: Koin documentation — Kotlin Multiplatform setup. https://insert-koin.io/docs/setup/koin
[^4]: Koin documentation — testing and `checkModules`/`verify`. https://insert-koin.io/docs/reference/koin-test/checkmodules
[^5]: Koin LTS (Kotzilla) — long-term support on stabilized versions such as Koin 3.5. https://kotzilla.io/koin-lts

## Tags

kotlin, dependency-injection, kotlin-multiplatform, android, di-framework, service-locator, ksp, jvm, compose-multiplatform, ktor
