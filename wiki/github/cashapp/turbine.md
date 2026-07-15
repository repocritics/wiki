# cashapp/turbine

> A small testing library for kotlinx.coroutines `Flow` that turns emissions, completion, and errors into events you pull and assert on.

[GitHub repo](https://github.com/cashapp/turbine) ·
[Official website](https://cashapp.github.io/turbine/docs/1.x/) ·
[License: Apache-2.0](https://github.com/cashapp/turbine/blob/trunk/LICENSE.txt)

## Overview

Turbine is a focused testing library for `kotlinx.coroutines` `Flow`, maintained by Cash App (Square/Block) and published under the `app.cash.turbine` Maven coordinate[^1]. Testing a cold or hot `Flow` directly is awkward: `collect` suspends indefinitely, emissions arrive concurrently with your assertions, and terminal events (completion, exceptions) are easy to miss. Turbine reframes a flow as a queue of discrete events — items, completion, error — that the test body pulls one at a time with `awaitItem()`, `awaitComplete()`, and `awaitError()`.

The defining design choice is that Turbine is **strict by default**: if your validation block finishes with events still in the queue, the test fails with an `Unconsumed events found` assertion[^2]. This catches emissions you did not expect and forces tests to account for the full lifecycle of the flow. The tradeoff is verbosity — you consume or explicitly ignore every event — in exchange for tests that fail loudly rather than passing on incomplete assertions.

Turbine is a Kotlin Multiplatform library, so the same API is available on JVM, Android, Native, and JS/Wasm targets. Internally a `Turbine` is a thin wrapper over a coroutines `Channel`; most of the public surface is implemented as extension functions on `Channel` and re-exposed through the narrower `Turbine` / `ReceiveTurbine` types[^2].

## Getting Started

```kotlin
// build.gradle.kts
repositories { mavenCentral() }
dependencies {
  testImplementation("app.cash.turbine:turbine:1.2.1")
}
```

```kotlin
import app.cash.turbine.test
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.test.runTest
import kotlin.test.Test
import kotlin.test.assertEquals

class ExampleTest {
  @Test fun emitsThenCompletes() = runTest {
    flowOf("one", "two").test {
      assertEquals("one", awaitItem())
      assertEquals("two", awaitItem())
      awaitComplete()
    }
  }
}
```

The `test` extension launches a coroutine that collects the flow into a `Turbine`, runs the validation block against the read-only `ReceiveTurbine` receiver, then cancels the collector and calls `ensureAllEventsConsumed()` for you.

## Architecture / How It Works

A `Turbine<T>` is a wrapper over a rendezvous `Channel<T>`. Producers (`Flow.collect` feeding the turbine, or manual `add()` calls on a standalone `Turbine`) push events in; the test body pulls them out with the `await*` family, which suspend until an event is ready. This "pull" model is the key inversion: instead of using `runCurrent()` to push a flow forward under a `TestScope`, the test parks on `awaitItem()` until the collector produces the next value, keeping the assertion code in the driver's seat[^2].

Three entry points cover the common shapes:

- **`Flow.test { }`** — collect a single flow, auto-cancel and verify at block end. The everyday path.
- **`Flow.testIn(scope)`** — return a `ReceiveTurbine` bound to a caller-supplied scope, for asserting across multiple concurrent flows inside a `turbineScope { }`. Unlike `test`, it cannot clean up its own collector; you must terminate it (`cancel()`, `awaitComplete()`, `awaitError()`) or run it on `runTest`'s `backgroundScope`, or the test hangs[^2].
- **`Turbine<T>()`** — a standalone queue with no flow attached, used to make fakes observable (for example capturing navigation calls in a `FakeNavigator`).

Terminal signals are first-class events, not exceptions. A flow that throws produces an `Error` event consumed via `awaitError()` — the exception does not propagate out of `collect` into your test. Failing to consume it surfaces as an `Unconsumed events found` error with the original throwable attached as the cause.

Standalone turbines also expose non-suspending `take*` variants (`takeItem()`, etc.) alongside the suspending `await*` methods, so mixed coroutine / non-coroutine code (for example RxJava `TestObserver` bridges) can drain the queue synchronously. These throw if called from a suspending context on the JVM[^2].

## Production Notes

- **Timeouts are wall-clock, not virtual.** Turbine's per-await timeout (default 3 seconds) uses real time and deliberately ignores `runTest`'s virtual clock[^2]. On loaded or slow CI machines a genuinely-correct test can flake with `No value produced in 3s`. Override per call with `test(timeout = ...)`, per turbine, or per block with `withTurbineTimeout { }` — but understand you are tuning against real machine speed, not test-scheduler time.
- **Unconsumed-events strictness is a feature, not a bug.** Migrations frequently break because a flow emits one more item (or a `Complete`) than the test consumes. Use `cancelAndIgnoreRemainingEvents()` or `expectMostRecentItem()` deliberately rather than reflexively; silencing the check hides real emissions.
- **Shared/hot flows are order-sensitive.** Calling `emit` on a `MutableSharedFlow(replay = 0)` before `collect` drops the value. Turbine guarantees the flow under test runs to its first suspension point before the block proceeds, so emitting *inside* the `test { }` block after `test` is entered is safe; emitting before is not[^2]. `StateFlow`/`SharedFlow` never complete, so tests must `cancel` rather than `awaitComplete`.
- **`testIn` hang risk.** The most common self-inflicted failure: a `testIn` turbine whose flow never terminates and is not on `backgroundScope`. The test hangs until the outer timeout. Prefer `runTest { turbineScope { ... testIn(backgroundScope) } }`.
- **Unstable transitive dependency.** Turbine's own API is stable, but it depends on `UnconfinedTestDispatcher` from the `kotlinx-coroutines-test` artifact, which the coroutines team marks unstable. Behavior can shift with coroutines upgrades until that API stabilizes (tracked in issue #132)[^3]. Pin coroutines and Turbine versions together and re-run the suite on upgrades.
- **Multiplatform native caveats.** Because the same code runs on Kotlin/Native, threading and dispatcher assumptions that hold on the JVM (for example the `take*` suspending-context guard) do not all behave identically across targets; validate flaky Native tests against dispatcher configuration first.

## When to Use / When Not

**Use when:**
- You test `kotlinx.coroutines` `Flow`, `StateFlow`, or `SharedFlow` and want lifecycle-complete assertions.
- You use the MVI/presenter pattern (Molecule, Circuit, Compose state) and assert on emitted UI models.
- You need fakes whose interactions are observable from a test as an event queue.
- You want a test to fail when a flow emits more than you asserted.

**Avoid when:**
- You are not on Kotlin coroutines `Flow` (use RxJava's `TestObserver`, Project Reactor's `StepVerifier`, etc.).
- Your flow logic is trivially synchronous and `flow.toList()` under `runTest` already reads clearly — Turbine adds a dependency for little gain.
- You need virtual-time control of the *awaiting* side; Turbine's wall-clock timeout does not participate in `runTest`'s time skipping.

## Alternatives

- Kotlin/kotlinx.coroutines — the built-in `runTest` + `Flow.toList()` / `TestScope` covers simple collection with no extra dependency; reach for it when you do not need event-by-event or terminal assertions.
- kotest/kotest — has flow-testing matchers and a broader assertion DSL; use when you already standardize on Kotest and want one framework.
- ReactiveX/RxJava — `TestObserver`/`TestSubscriber` is the equivalent for Rx types; use it when your streams are RxJava, not Flow.
- reactor/reactor-core — `StepVerifier` is the JVM-Reactive-Streams analog; use for Project Reactor `Flux`/`Mono`.
- cashapp/molecule — complementary, not a replacement: it builds flows from Compose, which you then assert with Turbine.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-07-29 | Project started at Cash App[^1]. |
| 0.x | 2020–2022 | Early single-flow `test { }` API; JVM-focused. |
| 1.0.0 | 2022 | Stable API; Kotlin Multiplatform; `testIn` / `turbineScope`, standalone `Turbine`[^2]. |
| 1.2.1 | 2026 (current) | Latest published release on Maven Central[^1]. |
| 1.3.0-SNAPSHOT | in dev | Development snapshots in the Central Portal snapshots repo[^1]. |

## References

[^1]: Turbine README and downloads — coordinate `app.cash.turbine:turbine`, current release 1.2.1. https://github.com/cashapp/turbine
[^2]: Turbine usage documentation — `test`/`testIn`, event consumption, shared-flow ordering, timeouts, standalone turbines. https://cashapp.github.io/turbine/docs/1.x/
[^3]: Turbine issue #132 — dependency on the unstable `UnconfinedTestDispatcher` API. https://github.com/cashapp/turbine/issues/132

## Tags

kotlin, kotlin-multiplatform, coroutines, flow, testing, testing-library, android, jvm, reactive, cashapp
