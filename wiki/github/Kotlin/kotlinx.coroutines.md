# Kotlin/kotlinx.coroutines

> JetBrains' official coroutines library for Kotlin — structured concurrency, dispatchers, and cold `Flow` streams on top of the compiler's `suspend` primitive.

[GitHub repo](https://github.com/Kotlin/kotlinx.coroutines) ·
[Coroutines guide](https://kotlinlang.org/docs/coroutines-guide.html) ·
[License: Apache-2.0](https://github.com/Kotlin/kotlinx.coroutines/blob/master/LICENSE.txt)

## Overview

`kotlinx.coroutines` is the runtime library that turns Kotlin's language-level `suspend`
functions into a usable concurrency model. The compiler provides the low-level machinery —
`suspend` functions compile to a continuation-passing state machine — but on its own the
language ships almost no runtime: no dispatchers, no `launch`/`async`, no `Flow`, no
cancellation propagation. This library provides all of that[^1]. It is developed by JetBrains
alongside the compiler; each release is a "companion version" for a specific Kotlin release
(1.11.0 targets Kotlin 2.2.20), and its own versioning is decoupled from the compiler's.

The defining idea is **structured concurrency**: coroutines are launched into a
`CoroutineScope` and form a parent-child `Job` tree. A parent does not complete until its
children complete, cancelling a parent cancels all children, and an uncaught failure in a
child cancels the parent (unless a `SupervisorJob` breaks that link). This makes coroutine
lifetimes lexically scoped and leak-resistant — the opposite of fire-and-forget threads or
raw callbacks. The design was led largely by Roman Elizarov and predates comparable
"structured concurrency" work in other ecosystems[^2].

The central tension is that `suspend` is a **function color**: suspend functions can only be
called from other suspend functions or a coroutine builder. Bridging to and from blocking
code (`runBlocking`, `Dispatchers.IO`) is where most production incidents originate. Coroutines
are cheap (a suspended coroutine is roughly an object, not a thread), but the cooperative
cancellation and exception-propagation rules are subtle and are the main source of bugs.

## Getting Started

Add the core artifact (Gradle Kotlin DSL); add the `-android` artifact for `Dispatchers.Main`
on Android:

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")
    // implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.11.0")
}
```

```kotlin
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.seconds

suspend fun main() = coroutineScope {          // structured scope: waits for both children
    val a = async { fetch("a") }               // Deferred<T>, runs concurrently
    val b = async { fetch("b") }
    println(a.await() + b.await())

    launch {
        delay(1.seconds)                        // suspends, does not block the thread
        println("done")
    }
}

suspend fun fetch(id: String): String = withContext(Dispatchers.IO) {
    // blocking or I/O-bound work goes on the IO dispatcher, not Default/Main
    "$id-result"
}
```

## Architecture / How It Works

**Continuations.** The compiler rewrites each `suspend` function into a state machine
implementing `Continuation<T>`. At every suspension point the coroutine can yield its thread;
resumption calls back into the state machine with the result. This is why a suspended
coroutine costs memory but not a thread — thousands of concurrent coroutines can share a small
thread pool.

**CoroutineContext.** Every coroutine carries an indexed set of elements: its `Job`, a
`CoroutineDispatcher`, an optional `CoroutineExceptionHandler`, a `CoroutineName`, and
arbitrary `ThreadLocal`-backed elements (`ThreadContextElement`, e.g. SLF4J `MDCContext`).
Contexts combine with `+` and are inherited by children, with overrides.

**Dispatchers** decide which thread a coroutine runs on:
- `Dispatchers.Default` — CPU-bound work, backed by a shared pool sized to the number of cores.
- `Dispatchers.IO` — blocking I/O; elastic, defaults to up to 64 threads, and shares threads
  with `Default` from a common pool. `limitedParallelism(n)` carves out a view with bounded
  concurrency.
- `Dispatchers.Main` — single UI thread; requires a platform artifact (`-android`, JavaFX,
  Swing) or it throws at first use. `Main.immediate` skips re-dispatch when already on Main.
- `Dispatchers.Unconfined` — runs in the caller thread until first suspension; mostly for
  tests and advanced use.

**Flow** is a cold asynchronous stream: nothing runs until a terminal operator (`collect`)
subscribes, and each collection re-executes the producer. `flowOn` changes upstream context;
`buffer`/`conflate` decouple producer and consumer speed. `StateFlow` and `SharedFlow` are the
hot variants — `StateFlow` always has a current value and conflates via equality (intermediate
values can be dropped), `SharedFlow` supports configurable replay and buffering.

**Channels, `Mutex`, `Semaphore`, and `select`** provide CSP-style communication and
suspending synchronization primitives that never block an underlying thread.

**Multiplatform.** Core is published for JVM, JS, Wasm/JS, and every supported Kotlin/Native
target. On Native the new memory model (default since Kotlin 1.7.20) removed the old freezing
restrictions, so multithreaded coroutines behave as on the JVM.

## Production Notes

**`runBlocking` blocks a real thread.** It is the bridge from blocking code into suspend code
and is correct in `main`, tests, and some framework glue — but calling it on a request thread,
UI thread, or inside a coroutine defeats the point and can deadlock a bounded pool. Treat it as
a boundary tool, never as "how I call a suspend function from here."

**Cancellation is cooperative.** A coroutine is only cancelled at a suspension point that
checks for it. A tight CPU loop with no `suspend` call will ignore cancellation forever; insert
`ensureActive()` or `yield()`. Critically, cancellation works by throwing
`CancellationException` on resume — a blanket `try { ... } catch (e: Exception)` that swallows
it silently breaks structured concurrency. Rethrow `CancellationException` (or catch specific
types). Cleanup that must run during cancellation needs `withContext(NonCancellable)`.

**Exception propagation differs by builder.** `launch` propagates an uncaught exception
*immediately* up the Job tree; `async` stores it and rethrows on `await()`. A
`CoroutineExceptionHandler` only fires for `launch`-style root coroutines — putting one in an
`async` child, or below a `coroutineScope`, does nothing. Use `SupervisorJob`/`supervisorScope`
when one child's failure should not cancel its siblings (common in UI and server request
scopes).

**`GlobalScope` leaks.** Coroutines launched in `GlobalScope` are unstructured — they are not
tied to any lifecycle and are not cancelled when the surrounding work ends. It is marked
`@DelicateApi`. Prefer a scope bound to a real lifecycle (`viewModelScope`, `lifecycleScope`,
a server request scope, or an explicit `CoroutineScope` you cancel).

**Testing.** `kotlinx-coroutines-test` provides `runTest`, which runs on virtual time so
`delay` completes instantly and the scheduler is controllable. `StandardTestDispatcher` queues
coroutines until you advance the clock; `UnconfinedTestDispatcher` runs eagerly. Override
`Dispatchers.Main` with `Dispatchers.setMain()` in `@Before`. Tests that inject a hardcoded
`Dispatchers.IO` instead of an injectable dispatcher are the usual reason `runTest`'s virtual
clock "doesn't work."

**Debugging.** Suspension hides call stacks. Enable stacktrace recovery and coroutine names
with `-Dkotlinx.coroutines.debug`, or use `DebugProbes` from `kotlinx-coroutines-debug` to dump
live coroutines; there is measurable overhead, so keep it out of hot production paths. On
Android, exclude `DebugProbesKt.bin` from the APK to avoid shipping the debug resource.

## When to Use / When Not

**Use when:**
- You are on Kotlin and need async I/O, concurrency, or reactive-style streams — this is the
  standard, first-party answer.
- You want structured, cancellable concurrency with lifecycles (Android `ViewModel`, Ktor
  request handling, Compose).
- You need one concurrency model that works across JVM, Android, Native, JS, and Wasm.

**Avoid / reconsider when:**
- You are calling from Java only — the API is designed for Kotlin syntax; from Java, the
  `Future`/reactive integration modules or plain JVM concurrency are more ergonomic.
- Your workload is pure blocking JDBC/file I/O on the JVM and you would rather not deal with
  function coloring — JDK 21 virtual threads offer blocking-style code without `suspend`.
- The team is new to Kotlin and the cancellation/exception model would be adopted without care;
  the footguns above cause silent bugs.

## Alternatives

- reactor/reactor-core — reactive streams for the JVM; use it when you're in Spring WebFlux or
  need a large operator set with backpressure and Java interop (bridged via `kotlinx-coroutines-reactor`).
- ReactiveX/RxJava — mature reactive library; use when maintaining an existing Rx codebase or
  targeting Java, with `kotlinx-coroutines-rx3` to interop.
- Kotlin/kotlinx-coroutines' own `Flow` — use instead of an external reactive library when the
  project is Kotlin-first and you want cold streams without a second dependency.
- openjdk/jdk virtual threads (Project Loom, JDK 21+) — use when JVM-only and you prefer
  blocking style over colored `suspend` functions.
- arrow-kt/arrow (Arrow Fx) — use when you want typed functional effects and error handling
  layered on top of coroutines.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016-08 | First public release, tracking experimental coroutines in Kotlin 1.1[^3]. |
| 0.26 | 2018-09 | Structured concurrency introduced (`CoroutineScope`, scope builders)[^2]. |
| 1.0.0 | 2018-10 | Stable release alongside Kotlin 1.3, which stabilized `suspend`[^1]. |
| 1.3.0 | 2019-08 | `Flow` API stabilized. |
| 1.4.x | 2020 | `StateFlow`/`SharedFlow`, reworked exception handling. |
| 1.6.x | 2022 | `Dispatchers.IO.limitedParallelism`, native improvements. |
| 1.11.0 | 2026 | Companion release for Kotlin 2.2.20[^4]. |

## References

[^1]: kotlinx.coroutines README and "Guide to kotlinx.coroutines by example." https://kotlinlang.org/docs/coroutines-guide.html
[^2]: Roman Elizarov, "Structured concurrency" — 2018-09-11. https://elizarov.medium.com/structured-concurrency-722d765aa952
[^3]: Coroutines Design Document (KEEP). https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md
[^4]: kotlinx.coroutines changelog. https://github.com/Kotlin/kotlinx.coroutines/blob/master/CHANGES.md

## Tags

kotlin, coroutines, async, concurrency, structured-concurrency, flow, reactive-streams, jvm, kotlin-multiplatform, android, dispatchers, jetbrains
