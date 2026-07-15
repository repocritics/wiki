# cashapp/molecule

> Build a StateFlow or Flow stream by running Jetpack Compose's compiler and runtime with no UI attached.

[GitHub repo](https://github.com/cashapp/molecule) ·
[Official docs](https://cashapp.github.io/molecule/docs/2.x/) ·
[License: Apache-2.0](https://github.com/cashapp/molecule/blob/trunk/LICENSE.txt)

## Overview

Molecule is a small library from Cash App (Block) that repurposes the Jetpack
Compose compiler and runtime as a general-purpose state-management engine,
disconnected from Compose UI[^1]. You write a `@Composable` function that returns
a model object, and Molecule runs it as a coroutine that emits a new value —
into a `StateFlow` or `Flow` — every time recomposition produces a different
result. The pitch, stated bluntly in the README's meme image, is "not a
framework, just a headless Compose."

The problem it targets is presenter/view-model logic. Cash App's presenters
traditionally exposed a single stream of display models via `Flow` or RxJava
`Observable`, and combining several reactive sources (`combine`, `onStart`,
`flatMapLatest`) scales non-linearly in complexity[^1]. Molecule lets you write
that same combination as ordinary imperative Kotlin — `val x by
flow.collectAsState(...)`, `if/else`, `remember`, `LaunchedEffect` — and get a
reactive stream out the other end. It also solves a layering nuisance:
`launchMolecule` returns a `StateFlow` whose initial value is computed
synchronously by the presenter, so the view no longer has to invent a default
for `collectAsState`.

The defining tradeoff: you inherit all of Compose's mental model (recomposition,
snapshot state, frame clocks, effect lifecycles) for code that has nothing to do
with UI, and you tie your build to the Compose compiler plugin — a large
conceptual surface for what is otherwise a stream transformer. In return you get
Compose's ergonomics (fine-grained state tracking, `remember`, structured
effects) in your presentation logic.

## Getting Started

Molecule requires JetBrains' Kotlin Compose compiler plugin applied to any
module that calls `launchMolecule` or defines Molecule composables[^2]. Then add
the runtime dependency:

```groovy
dependencies {
  implementation("app.cash.molecule:molecule-runtime:2.2.0")
}
```

```kotlin
// A presenter is a @Composable that returns a model, not UI.
@Composable
fun ProfilePresenter(
  userFlow: Flow<User>,
  balanceFlow: Flow<Long>,
): ProfileModel {
  val user by userFlow.collectAsState(null)
  val balance by balanceFlow.collectAsState(0L)
  return if (user == null) Loading else Data(user.name, balance)
}

// Run it into a StateFlow, tied to a CoroutineScope.
val models: StateFlow<ProfileModel> = scope.launchMolecule(mode = ContextClock) {
  ProfilePresenter(db.users(), db.balances())
}
```

Use `moleculeFlow(mode = Immediate) { ... }` when you want a cold `Flow` instead
of a hot `StateFlow`.

## Architecture / How It Works

Compose is, at its core, a compiler and runtime for tracking state and mutating
a tree of nodes — the UI toolkit is just one consumer of it. Molecule "just"
glues that runtime to kotlinx.coroutines, driving recomposition and reading the
returned value instead of managing a node tree[^1]. Under the hood it starts a
Compose `Recomposer`, composes your function, and on each recomposition publishes
the function's return value to a flow.

Because it is genuinely Compose underneath, recomposition is gated by a
**frame clock** — Compose never does work until a `MonotonicFrameClock` in the
coroutine context signals the next frame. Molecule surfaces this as a required
`RecompositionMode` argument on every entry point[^3]:

- `RecompositionMode.ContextClock` fishes a `MonotonicFrameClock` out of the
  calling `coroutineContext` and throws if none exists. On Android, running on
  `AndroidUiDispatcher.Main` gives you a clock synchronized to the device frame
  rate, so the molecule recomposes in lock-step with the display.
- `RecompositionMode.Immediate` builds an immediate clock that produces a frame
  whenever the enclosing flow is ready to emit. It needs no clock wiring, which
  makes it the mode for unit tests and for running molecules off the main thread.

The choice matters: `ContextClock` coalesces multiple state changes within a
frame into one emission (throttled to frame rate); `Immediate` can emit on every
change. The same presenter can produce different emission counts under the two
modes.

Molecule 2.x is a Kotlin/Compose Multiplatform library — the runtime is not
Android-specific and the state logic can run on any Kotlin target the Compose
runtime supports. Testing is done with Cash App's own Turbine library:
`moleculeFlow(Immediate) { ... }.test { awaitItem() }` behaves like any other
flow under Turbine[^4].

## Production Notes

- **Frame-clock behavior is the top footgun.** `ContextClock` requires a clock
  in context or it throws at runtime; and it batches emissions to frame
  boundaries, so a test or consumer expecting one emission per state change will
  see fewer under `ContextClock` than under `Immediate`. Pick `Immediate` for
  deterministic tests.
- **The Compose compiler plugin is a hard dependency of your build**, not just
  Molecule's. Every module defining Molecule composables must apply it, and the
  plugin version is coupled to your Kotlin version — Compose-compiler/Kotlin
  version skew is the usual source of build breakage after a Kotlin upgrade[^2].
- **`LaunchedEffect` runs forever until the scope is cancelled.** A presenter's
  effects live as long as the `launchMolecule` scope; leaking the scope leaks the
  coroutine. Tie the scope to the consuming lifecycle deliberately.
- **`StateFlow` conflation can hide intermediate states.** `launchMolecule`
  returns a `StateFlow`, which drops values a slow collector doesn't observe. If
  every emission matters, use `moleculeFlow` and collect it directly.
- **Not a rendering framework, and Compose bugs are your bugs.** Molecule
  produces model streams; you still render them with Compose UI, Views, or
  SwiftUI. Defects show up as unexpected recomposition, stale `remember`, or
  effects re-keying — the same class Compose UI developers hit, minus the visual
  feedback that usually makes them obvious.

## When to Use / When Not

**Use when:**
- You already build UI with Compose and want your presenters written in the same
  idiom (imperative state, `remember`, effects) instead of RxJava/Flow operator
  chains.
- You have presentation logic that combines several reactive sources and the
  `combine`/`flatMapLatest` ceremony has become hard to read.
- You want a `StateFlow<Model>` with a synchronously-available initial value for
  a Compose UI consumer.

**Avoid when:**
- You are not otherwise using Compose — pulling in the Compose compiler plugin
  and runtime purely for stream transformation is a heavy dependency.
- Your logic is simple enough that plain `Flow`/`combine` is clear; Molecule adds
  a recomposition mental model you must carry.
- You need strict, non-conflated emission of every intermediate state and don't
  want to reason about frame clocks.

## Alternatives

- cashapp/turbine — companion, not competitor: testing library for flows,
  used to test Molecule presenters; use for asserting flow emissions.
- Kotlin `kotlinx.coroutines` `Flow` / `StateFlow` — use directly when your
  combination logic is simple and you want zero Compose coupling.
- badoo/Reaktive or ReactiveX/RxJava — use when your codebase is already
  reactive-operator-centric and you don't want the Compose runtime in presenters.
- arkivanov/MVIKotlin — use when you want a full MVI architecture with stores and
  time-travel rather than a bare model-stream transformer.
- slackhq/circuit — use when you want an opinionated Compose-based
  presenter+UI architecture with navigation, not just the presenter stream.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021-11-10 | First public release[^5]. |
| 0.5.0 | 2022-10-13 | Pre-1.0 API iteration. |
| 1.0.0 | 2023-07-19 | First stable release[^5]. |
| 1.3.0 | 2023-10-31 | Continued 1.x line. |
| 2.0.0 | 2024-05-28 | Compose Multiplatform runtime line[^5]. |
| 2.1.0 | 2025-04-12 | 2.x maintenance. |
| 2.2.0 | 2025-09-24 | Latest stable; 2.3.0-SNAPSHOT in development[^2]. |

Actively maintained: last commit to `trunk` on 2026-07-04, ~2.2k stars, authored
originally by Jake Wharton and maintained under Cash App's open-source org.

## References

[^1]: Molecule README — "Build a StateFlow or Flow stream using Jetpack Compose" and Introduction. https://github.com/cashapp/molecule/blob/trunk/README.md
[^2]: Molecule README — "Usage" (Compose compiler plugin requirement, dependency coordinates, snapshot repo). https://github.com/cashapp/molecule#usage
[^3]: Molecule README — "Frame Clock" (`RecompositionMode.ContextClock` vs `Immediate`). https://github.com/cashapp/molecule#frame-clock
[^4]: Molecule README — "Testing" with Turbine. https://github.com/cashapp/turbine/
[^5]: Molecule GitHub releases (tag dates via GitHub API). https://github.com/cashapp/molecule/releases

## Tags

kotlin, jetpack-compose, state-management, coroutines, stateflow, android, compose-multiplatform, presenter, reactive, cash-app
