# mockk/mockk

> A Kotlin-first mocking library that mocks final classes and coroutines out of the box, at the cost of bytecode magic.

[GitHub repo](https://github.com/mockk/mockk) ·
[Official website](https://mockk.io) ·
[License: Apache-2.0](https://github.com/mockk/mockk/blob/master/LICENSE)

## Overview

MockK is a mocking library written for Kotlin, first published around 2017–2018 by Oleksiy Pylypenko[^1]. It exists because the dominant JVM mocking tool, Mockito, was designed for Java and fits Kotlin awkwardly: Kotlin classes and methods are `final` by default, Kotlin has `object` singletons, extension functions, and `suspend` coroutines — none of which map cleanly onto Mockito's proxy model without extra opt-ins. MockK's pitch is that these are first-class: you can `mockk<FinalClass>()` with no `open` keyword and stub a `suspend fun` with `coEvery { }` directly.

The DSL is the visible surface — `every { }`, `verify { }`, `mockk`, `spyk`, `slot`, `capture` — but the defining tension is what happens underneath. To mock final types MockK relies on bytecode instrumentation (a JVM agent / ByteBuddy-style class redefinition, and a subclassing agent for Android/Dalvik). That mechanism is what makes the ergonomics possible and also what makes MockK the more fragile choice: it interacts badly with the JDK module system on newer JVMs, is slower than a plain proxy library, and produces failure modes (`InaccessibleObjectException`, class-cast errors on generic returns) that have nothing to do with your test logic.

As of 2026 MockK is the de facto standard for testing Kotlin code and is used across most Android and Kotlin-backend codebases. It is actively maintained — releases land every few weeks and the latest tag is v1.14.11 (May 2026)[^2] — but development is incremental; there has been no 2.x reset of the API.

## Getting Started

```kotlin
// build.gradle.kts
testImplementation("io.mockk:mockk:1.14.11")
// Android unit tests use a separate artifact:
// testImplementation("io.mockk:mockk-android:1.14.11")
```

```kotlin
import io.mockk.*

class Car { fun drive(d: Direction): Outcome = TODO() }

@Test
fun example() {
    val car = mockk<Car>()                              // strict by default
    every { car.drive(Direction.NORTH) } returns Outcome.OK

    assertEquals(Outcome.OK, car.drive(Direction.NORTH))

    verify { car.drive(Direction.NORTH) }               // was it called?
    confirmVerified(car)                                // nothing else was
}
```

Mocks are **strict**: calling an unstubbed method throws `MockKException`. `mockk<T>(relaxed = true)` returns default values instead, and `relaxUnitFun = true` relaxes only `Unit`-returning functions — a common middle ground.

## Architecture / How It Works

MockK stubs a call in two phases. `every { car.drive(NORTH) }` runs the block in a special recording mode: the mock records the invocation and the argument matchers rather than executing anything, and the trailing `returns` / `answers` binds an answer to that recorded signature. `verify { }` replays the recorded call log against the same matcher machinery. Argument matchers (`any()`, `eq()`, `match { }`, `slot()` + `capture()`) are the core abstraction — they are objects captured during recording, not values.

The interesting part is how the mock object is produced:

- **JVM.** MockK installs a Java agent (`mockk-agent`) that uses class redefinition / ByteBuddy to intercept methods on final classes and objects. This is why no `open` is required — the difference from Mockito's historical need for `mockito-inline`.
- **Android.** Dalvik/ART cannot redefine loaded classes the same way, so `mockk-android` ships a subclassing agent. Instrumented (on-device) tests additionally need `mockk-agent`.
- **`mockkObject` / `mockkStatic` / `mockkConstructor`.** These transform an existing singleton, a static/top-level function set, or a constructor into a mock in place. They mutate global state for the duration of the test and **must be reverted** with `unmockkObject` / `unmockkStatic` / `unmockkAll`, or the instrumentation leaks into later tests in the same JVM.
- **Coroutines.** `coEvery` / `coVerify` are the `suspend` counterparts; internally the suspend call is recorded like any other but through a coroutine-aware invocation path.

Because behavior is keyed on recorded matchers, a `verify` block must use matchers that are *equal* to the ones used in `every`. This bites hardest with `mockkConstructor`, where each distinct set of `constructedWith(...)` matchers creates its own prototype mock — mismatched matchers between stub and verify silently target different prototypes.

## Production Notes

- **JDK 16+ friction.** Spies, `mockkStatic`, and `mockkObject` can throw `InaccessibleObjectException` / `IllegalAccessException` on JDK 16 and later because of strong encapsulation of JDK internals. The documented workaround is adding `--add-opens` JVM args to the test task[^3]. This is the single most common production complaint.
- **Global-state footguns.** `mockkObject`/`mockkStatic`/`mockkConstructor` are process-wide. Forgetting to unmock — or an exception skipping the teardown — corrupts unrelated tests with confusing failures. The JUnit5 `MockKExtension` calls `unmockkAll` + `clearAllMocks` in `@AfterAll` to mitigate this; the JUnit4 `MockKRule` and manual `MockKAnnotations.init` do not clean statics for you.
- **Parallel tests.** `clearAllMocks()` is not thread-safe by default (`currentThreadOnly=false`). Running test classes in parallel with the extension's default teardown can race; use `clearAllMocks(currentThreadOnly = true)` or the `RequireParallelTesting` opt-in (recent versions).
- **Performance.** MockK is meaningfully slower to create mocks than Mockito because of the instrumentation and DSL recording overhead. On large suites (thousands of tests) this is visible; teams sometimes reserve `mockkStatic`/`mockkConstructor` for the few tests that truly need them.
- **Known hard limits.** Inline functions cannot be mocked (they are inlined at the call site, so there is no method to intercept). Using a spy with a `suspend` function gives unexpected results[^4]. Relaxed mocks with generic return types tend to throw class-cast exceptions — stub those explicitly. PowerMock needs a workaround to coexist.
- **Verification hygiene.** `confirmVerified`, `checkUnnecessaryStub`, and the `@MockKExtension.ConfirmVerification` / `CheckUnnecessaryStub` annotations catch over-stubbing, but note IDE test runners may not surface the thrown exception — Gradle is what enforces it.

## When to Use / When Not

**Use when:**
- The code under test is idiomatic Kotlin — final classes, `object`s, `suspend` functions, extension functions.
- You want mocking without sprinkling `open` or configuring `mockito-inline`.
- You need coroutine mocking (`coEvery`/`coVerify`) with a native DSL.

**Avoid when:**
- The codebase is primarily Java — Mockito is faster, more stable, and better documented there.
- You run on locked-down JVMs where adding `--add-opens` is not acceptable.
- You lean heavily on static/constructor mocking in a huge suite and cannot absorb the speed and global-state cost — prefer refactoring to injected dependencies.

## Alternatives

- mockito/mockito — the JVM incumbent; pair with mockito-kotlin for Kotlin ergonomics. Use when the project is Java-first or you need maximum stability and speed.
- nhaarman/mockito-kotlin — Kotlin DSL wrapper over Mockito; use when you want Mockito's engine but Kotlin-friendly syntax and don't need final-class mocking without `mockito-inline`.
- Ninja-Squad/springmockk — replaces Spring Boot's `@MockBean`/`@SpyBean` with MockK equivalents; use when a Spring Boot Kotlin app needs MockK in the context.
- quarkiverse/quarkus-mockk — mock Quarkus beans with MockK; use in Quarkus + Kotlin services.
- kotest/kotest — a test framework (not a mocker) that integrates with MockK; use when you also want Kotlin-native assertions/spec styles alongside mocking.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | ~2018 | First stable line; Kotlin-native mocking of final classes via JVM agent[^1]. |
| 1.12.x | 2021–2022 | Constructor/static mocking maturation, Android agent improvements. |
| 1.13.0 | 2022-09 | Baseline moves to Kotlin 1.4+ support[^5]. |
| 1.13.11 | 2024-05 | `MockKExtension.keepmocks` property; teardown controls. |
| 1.14.0 | 2025-04 | Continued 1.14 line; regular maintenance cadence. |
| 1.14.11 | 2026-05 | Latest release as of mid-2026[^2]. |

## References

[^1]: MockK repository and history, mockk/mockk. https://github.com/mockk/mockk
[^2]: MockK releases (v1.14.11, 2026-05-29). https://github.com/mockk/mockk/releases
[^3]: MockK docs, "JDK 16+ access exceptions." https://github.com/mockk/mockk/blob/master/doc/md/jdk16-access-exceptions.md
[^4]: MockK issue #554, spy with suspending function. https://github.com/mockk/mockk/issues/554
[^5]: MockK README, "Kotlin version support" (Kotlin 1.4+ from 1.13.0). https://github.com/mockk/mockk

## Tags

kotlin, mocking, testing, unit-testing, jvm, android, coroutines, test-doubles, tdd, jvm-agent, mockk
