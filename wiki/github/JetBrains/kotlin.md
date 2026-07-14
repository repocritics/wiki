# JetBrains/kotlin

> A statically-typed, JVM-first language with pragmatic Java interop and a single toolchain that also targets native, JS, and WebAssembly.

[GitHub repo](https://github.com/JetBrains/kotlin) ·
[Official website](https://kotlinlang.org) ·
[License: Apache-2.0](https://github.com/JetBrains/kotlin/blob/master/license/README.md)

## Overview

Kotlin is a general-purpose language designed by JetBrains, first announced in 2011 and reaching 1.0 in February 2016[^1]. Its original pitch was narrow and concrete: a better Java for the JVM — null-safety in the type system, less boilerplate, and 100% two-way interop so it could be adopted file-by-file inside existing Java codebases. That interop-first design is why it spread through Android and server-side Java shops rather than as a greenfield language.

The defining tension in Kotlin is between that pragmatic JVM heritage and its later ambition as a **multiplatform** language. On the JVM it is mature, fast to adopt, and effectively the default for new Android development after Google's 2017 first-class support and 2019 "Kotlin-first" stance[^2]. Off the JVM — Kotlin/Native (LLVM), Kotlin/JS, and Kotlin/Wasm — the story is younger, the tooling thinner, and the debugging experience noticeably rougher. A team evaluating Kotlin is really evaluating two different maturity levels depending on target.

The repository here is the compiler, standard library, and the Gradle/Maven build plugins — not the language spec or the IntelliJ plugin (which lives in `JetBrains/intellij-community`). The compiler underwent a full frontend rewrite ("K2", the FIR-based frontend) that stabilized in Kotlin 2.0 in 2024[^3], the largest internal change in the project's history.

## Getting Started

Kotlin is almost always used through a build tool rather than the raw compiler. With Gradle (Kotlin DSL):

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "2.2.0"
    application
}
application { mainClass.set("MainKt") }
```

```kotlin
// src/main/kotlin/Main.kt
data class User(val id: Int, val name: String)

fun greet(user: User): String = "Hello, ${user.name}"

fun main() {
    val users = listOf(User(1, "Tom"), User(2, "Brad"))
    users.filter { it.id > 0 }
         .forEach { println(greet(it)) }
}
```

Run with `./gradlew run`. For quick experiments the standalone compiler (`kotlinc`, via SDKMAN or Homebrew) gives a REPL and `kotlinc Main.kt -include-runtime -d app.jar`.

## Architecture / How It Works

Kotlin is a single language with multiple backends, joined by a shared frontend:

1. **Frontend (K2 / FIR)** — parses source, resolves names, runs type inference and diagnostics, producing a resolved semantic tree. This is the part rewritten for 2.0; the old frontend is now removed. K2 is both faster and more consistent across the language's many features (smart casts, contracts, builder inference) that the legacy frontend handled with accumulated special cases.
2. **IR backends** — the frontend lowers to a common Intermediate Representation, and per-target backends emit output:
   - **Kotlin/JVM** — Java bytecode (the primary, most optimized target).
   - **Kotlin/Native** — LLVM bitcode → native binaries, used for iOS and other platforms without a JVM.
   - **Kotlin/JS** — JavaScript, with an IR-based backend that replaced the older direct one.
   - **Kotlin/Wasm** — WebAssembly, the newest and least stable target.

**Java interop** is the JVM backend's core value and its sharpest edge. Kotlin types compile to ordinary JVM signatures, so calls go both ways with no wrappers. But Java has no nullability in its type system, so values crossing from Java arrive as **platform types** (`String!`) that Kotlin does not null-check — a real NPE surface that null-safety otherwise eliminates.

**Coroutines** are a library (`kotlinx.coroutines`) built on a compiler feature: `suspend` functions are compiled into state machines via continuation-passing, so there is no OS thread per coroutine. Structured concurrency (`CoroutineScope`, `Job` hierarchies) is a convention the library enforces, not a language guarantee.

**Multiplatform (KMP)** shares a `commonMain` source set with `expect`/`actual` declarations resolved per target. It shares logic, not usually UI (Compose Multiplatform is a separate JetBrains project layered on top).

## Production Notes

**Compile times.** Kotlin compilation is meaningfully slower than `javac`, historically the top complaint on large JVM codebases. K2 (2.0+) improved frontend speed substantially; still, incremental compilation, the Gradle build cache, and configuration caching matter more than any single flag. Very large modules should be split — Gradle parallelism is per-module.

**Annotation processing: KAPT vs KSP.** KAPT runs Java annotation processors by generating Java stubs of your Kotlin, which is slow and stalls incremental builds. **KSP** (Kotlin Symbol Processing) reads Kotlin symbols directly and is the recommended path; Dagger/Hilt, Room, Moshi, and most modern libraries ship KSP support. Migrating off KAPT is one of the highest-leverage build-time wins.

**Coroutine footguns.** `GlobalScope` leaks work that outlives its caller — prefer a scoped `CoroutineScope`. Cancellation is cooperative: tight loops without a suspension point never cancel. Choosing the wrong `Dispatcher` (blocking IO on `Dispatchers.Default`) starves the shared thread pool. `runBlocking` in production request paths reintroduces the thread-blocking you adopted coroutines to avoid.

**Kotlin/Native maturity.** The rewritten memory manager (default since 1.7.20) removed the old thread-isolation restrictions that made early Native painful, but compile times are slow, binaries are large, and native-target debugging is weaker than JVM. Treat iOS/Native as a supported-but-younger platform, not JVM-equivalent.

**Version and binary compatibility.** The stdlib evolves conservatively but `@OptIn`/experimental APIs can change between minor releases. Kotlin and its Gradle plugin version are coupled to a supported Gradle range; mismatches surface as opaque build failures. Bumping the Kotlin version can shift compiler behavior (new warnings, stricter inference) even in a minor release.

**K2 migration.** Moving to 2.0's K2 frontend is mostly transparent for application code, but compiler plugins (Compose, kapt integrations, some KSP versions) and heavy metaprogramming needed aligned releases; teams on the bleeding edge hit plugin-compatibility gaps during the 2.0 transition.

## When to Use / When Not

**Use when:**
- You target the JVM or Android — it is the pragmatic default, with gradual adoption inside existing Java.
- You want null-safety, data classes, coroutines, and sealed hierarchies without leaving the Java ecosystem.
- You need to share business logic across Android and iOS and can accept KMP's younger tooling.
- You value one language spanning backend, Android, and (increasingly) web.

**Avoid when:**
- You need maximal build speed on a huge codebase and Java's simpler toolchain suffices.
- Your target is iOS-only — Swift is the native-first choice with better platform tooling.
- You want a language with a formal, vendor-independent standard and multiple mature implementations; Kotlin is effectively defined by JetBrains' compiler.
- Your web front-end needs are mainstream — Kotlin/JS and Wasm are viable but far outside the JS ecosystem's mass of tooling.

## Alternatives

- oracle/java (openjdk) — use plain Java when interop is one-directional, build simplicity dominates, or you don't need Kotlin's language features.
- scala/scala — use Scala when you want a richer functional type system on the JVM and can absorb its heavier learning curve and compile times.
- apple/swift — use Swift for iOS/macOS-first apps where native platform integration outweighs code sharing with Android.
- flutter/flutter (Dart) — use Flutter when the goal is a single shared UI across mobile, versus KMP's shared-logic/native-UI model.
- golang/go — use Go for backend services when operational simplicity and fast builds matter more than JVM-ecosystem depth.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-02 | First stable release; JVM-first, Java interop[^1]. |
| 1.1 | 2017-03 | Kotlin/JS, coroutines (experimental). |
| 1.3 | 2018-10 | Coroutines stable, multiplatform (experimental). |
| 1.4 | 2020-08 | New JVM/JS IR backends, tooling overhaul. |
| 1.5 | 2021-05 | Stable JVM IR backend, JVM records/sealed interfaces. |
| 1.7 | 2022-06 | K2 compiler alpha; new Kotlin/Native memory manager. |
| 1.9 | 2023-07 | K2 beta; Kotlin Multiplatform declared stable[^4]. |
| 2.0 | 2024-05 | K2 frontend stable — largest internal rewrite[^3]. |
| 2.2 | 2025 | Continued K2 refinement; Wasm and language features advancing. |

## References

[^1]: JetBrains, "Kotlin 1.0 Released: Pragmatic Language for the JVM and Android" — 2016-02-15. https://blog.jetbrains.com/kotlin/2016/02/kotlin-1-0-released-pragmatic-language-for-jvm-and-android/
[^2]: Google, "Announcing Kotlin on Android" (I/O 2017) and the 2019 Kotlin-first announcement. https://developer.android.com/kotlin
[^3]: JetBrains, "Kotlin 2.0.0 Released" (K2 compiler) — 2024-05-21. https://blog.jetbrains.com/kotlin/2024/05/kotlin-2-0-0-is-released/
[^4]: JetBrains, "Kotlin Multiplatform Is Stable and Production-Ready" — 2023. https://blog.jetbrains.com/kotlin/2023/11/kotlin-multiplatform-stable/

## Tags

kotlin, jvm, android, programming-language, compiler, multiplatform, coroutines, kotlin-native, static-typing, java-interop, gradle
