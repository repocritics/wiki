# pointfreeco/swift-composable-architecture

> A Redux/Elm-inspired unidirectional architecture for SwiftUI and UIKit, built around composable reducers, controlled dependencies, and exhaustive testing.

[GitHub repo](https://github.com/pointfreeco/swift-composable-architecture) ·
[Point-Free collection](https://www.pointfree.co/collections/composable-architecture) ·
[License: MIT](https://github.com/pointfreeco/swift-composable-architecture/blob/main/LICENSE)

## Overview

The Composable Architecture (TCA) is a state-management library for Apple-platform apps, developed by Brandon Williams and Stephen Celis of Point-Free and designed live across a long-running episode series[^1]. It models a feature as four pieces: a `State` value type, an `Action` enum enumerating everything that can happen, a `Reducer` that evolves state and returns effects, and a `Store` runtime that drives the loop. The pattern is unidirectional and value-typed, borrowing from Redux and The Elm Architecture but adapted to Swift's type system and Swift Concurrency.

TCA's central bet is that the ceremony pays for itself in testability and composition. Because state is a value type and effects are declared rather than performed inline, a `TestStore` can replay an entire user flow and assert on every intermediate state change and every effect feedback. Large features are assembled from small reducers via operators (`Scope`, `ifLet`, `forEach`), so isolated modules glue back together without losing that testability. As of 2026 it is the most-adopted third-party architecture in the iOS ecosystem, with roughly 14.8k stars and active maintenance[^2].

The defining tension is boilerplate and ecosystem gravity versus rigor. Compared to vanilla SwiftUI (`@Observable` + `@State`), TCA asks for more types, more indirection, and a large transitive dependency surface of Point-Free packages. In exchange you get controllable dependencies, exhaustive tests, and a uniform way to express side effects. Teams either find the discipline worth it or find it a tax on features that are mostly forms and navigation.

## Getting Started

Add the package in Xcode via **File → Add Package Dependencies** with `https://github.com/pointfreeco/swift-composable-architecture`, or in `Package.swift`:

```swift
dependencies: [
  .package(url: "https://github.com/pointfreeco/swift-composable-architecture", from: "1.0.0")
]
```

A minimal feature and its view:

```swift
import ComposableArchitecture
import SwiftUI

@Reducer
struct Counter {
  @ObservableState
  struct State: Equatable { var count = 0 }

  enum Action { case increment, decrement }

  var body: some Reducer<State, Action> {
    Reduce { state, action in
      switch action {
      case .increment: state.count += 1; return .none
      case .decrement: state.count -= 1; return .none
      }
    }
  }
}

struct CounterView: View {
  let store: StoreOf<Counter>
  var body: some View {
    HStack {
      Button("−") { store.send(.decrement) }
      Text("\(store.count)")
      Button("+") { store.send(.increment) }
    }
  }
}
```

Effects use Swift Concurrency via `.run`; dependencies are injected with the `@Dependency` property wrapper and overridden in tests through a `TestStore`'s `withDependencies:` closure.

## Architecture / How It Works

The runtime loop is: a view (or UIKit controller) calls `store.send(action)`; the `Store` runs the `Reducer`, which mutates `inout State` and returns an `Effect`; effects run asynchronously and can feed new actions back via `send`; state changes are observed and the UI re-renders.

- **Reducer as a protocol.** Since the pre-1.0 rewrite, a reducer is a type conforming to the `Reducer` protocol with a `body` built from other reducers, rather than a bare function[^3]. The `@Reducer` macro generates conformances, the `Action` case-path plumbing, and navigation glue. Composition happens by nesting child reducers inside a parent's `body` with `Scope`, `ifLet`, and `forEach`.
- **Observation.** As of 1.7 the library integrates Swift's Observation framework via `@ObservableState`, which lets SwiftUI views read `store.someField` directly and re-render on precise changes. This deprecated the older `ViewStore` / `WithViewStore` indirection that dominated earlier versions[^4]. On OS versions predating Observation, `swift-perception` backports the mechanism.
- **Dependencies.** Effects reach the outside world only through values registered with `swift-dependencies`. Each dependency has a `liveValue`, and optionally `testValue`/`previewValue`; unregistered access in tests triggers a failure, forcing explicit control. This is a separable package usable without the rest of TCA.
- **Case paths.** `Action` enums are navigated with `swift-case-paths`, giving key-path-like access into enum cases so reducers can pluck out and embed child actions.
- **Navigation & collections.** Tree-based navigation uses `@Presents` + `PresentationAction`; stack-based navigation uses `StackState`/`StackAction`. Collections of features use `IdentifiedArray` from `swift-identified-collections`.
- **Shared state.** The `@Shared` property wrapper (via `swift-sharing`) allows state to be shared across features and persisted to `appStorage`, `fileStorage`, or in-memory, without threading it through every parent.

Effects were historically Combine `Publisher`s; they are now built on `async`/`await`, with `combine-schedulers` and a `Clock`-based `TestClock` providing deterministic control over time in tests.

## Production Notes

**Macro and build cost.** `@Reducer`, `@ObservableState`, and CasePaths macros expand at compile time. Large modules with many reducers see noticeable increases in build and type-check time, and macro expansion has historically produced opaque diagnostics when a `State`/`Action` shape is wrong. Incremental builds mitigate this but the first clean build of a big TCA app is slow.

**Exhaustive testing is strict by design.** A `TestStore` fails if any state mutation is unasserted or any received effect action is left unhandled. This catches regressions but is brittle under refactors — a single new field forces test updates across flows. For integration-level tests where that granularity is noise, set `store.exhaustivity = .off` (optionally `.off(showSkippedAssertions: true)`).

**Dependency surface.** Adding TCA pulls in a large set of Point-Free packages (swift-dependencies, swift-case-paths, swift-perception, swift-navigation, swift-identified-collections, swift-custom-dump, combine-schedulers, swift-sharing, swift-clocks, and the issue-reporting overlay). This is by design — they are the seams that make testing work — but it is a meaningful transitive footprint and couples your app's build to Point-Free's release cadence.

**Performance.** The reducer runs on every action, and deeply nested composition means a leaf action still traverses parent reducers. Observation made view updates far more granular than the old `ViewStore` era, but very large `State` trees and high-frequency actions (e.g. per-keystroke, scroll, gesture streams) can still be a hotspot; profile before pushing all such state through the store. `Reducer._printChanges()` helps trace action volume.

**Migration churn.** Pre-1.0 the API changed frequently, including the reducer-function-to-protocol rewrite and the Combine-to-Concurrency effect transition. 1.0 (2023) stabilized the core, but 1.x still shipped large shifts — the Observation migration deprecated `ViewStore`/`WithViewStore`, and navigation and sharing APIs evolved. Upgrading a long-lived app across these is real work, though the library ships migration guides in its documentation.

**Deployment targets.** `@ObservableState`'s native path uses Swift's Observation (iOS 17+/equivalent); `swift-perception` backports it to older systems but adds its own rules (e.g. `WithPerceptionTracking` in some view contexts) that produce runtime warnings when missed.

## When to Use / When Not

**Use when:**
- The app has complex, interdependent state and business logic you want under test.
- You value deterministic, exhaustive tests of user flows including side effects and time.
- You're building modular features across a team and want a uniform composition story.
- You want controllable dependencies (network, clocks, UUIDs, storage) as a first-class concern.

**Avoid when:**
- The app is small, mostly presentational, or a set of simple forms — vanilla SwiftUI `@Observable` is far less ceremony.
- The team resists indirection or can't absorb the learning curve and Point-Free's idioms.
- You need a minimal dependency footprint or must avoid coupling to a single vendor's release cadence.
- Your bottleneck is high-frequency UI state (animations, gestures) that doesn't belong in a store.

## Alternatives

- ReSwift/ReSwift — a lighter Redux port for Swift; use when you want unidirectional data flow without TCA's effect system, dependency injection, macros, or exhaustive test harness.
- pointfreeco/swift-dependencies — TCA's dependency-injection library used standalone; use when you want controllable dependencies but not the whole store architecture.
- pointfreeco/swift-navigation — Point-Free's state-driven navigation tools without reducers; use when you like their navigation model but drive it from your own observable objects.
- Apple SwiftUI `@Observable` (built into the SDK) — the plain MV/MVVM path; use for most small-to-medium apps where the ceremony of a store isn't justified.
- CombineCommunity/CombineFeedback — a state-machine/feedback-loop library; use when you want unidirectional loops built on Combine rather than a reducer protocol.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2020-05 | Initial release; Combine-based effects, `ViewStore` indirection[^2]. |
| ~0.40 | 2022 | Reducer becomes a protocol; effects move toward Swift Concurrency[^3]. |
| 1.0.0 | 2023-07 | Stable API; `@Dependency`, `.run` effects, `TestStore`[^1]. |
| 1.4 | 2023 | `@Reducer` macro introduced. |
| 1.7 | 2024 | Observation integration via `@ObservableState`; `ViewStore` deprecated[^4]. |
| 1.10 | 2024 | `@Shared` state and the sharing tools. |

## References

[^1]: Point-Free, "Composable Architecture" collection and episode series. https://www.pointfree.co/collections/composable-architecture
[^2]: GitHub repository metadata (stars, forks, creation date 2020-05-03, last pushed 2026-07-09). https://github.com/pointfreeco/swift-composable-architecture
[^3]: Point-Free, "Reducer Protocol" — the migration from reducer functions to the `Reducer` protocol. https://www.pointfree.co/blog/posts/81-announcing-the-reducer-protocol
[^4]: Composable Architecture documentation, "Migrating to 1.7" / Observation. https://swiftpackageindex.com/pointfreeco/swift-composable-architecture/main/documentation/composablearchitecture/migratingto1.7

## Tags

swift, swiftui, uikit, state-management, unidirectional-data-flow, redux, architecture, reducer, testing, dependency-injection, ios, point-free
