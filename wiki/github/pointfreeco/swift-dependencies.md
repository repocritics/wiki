# pointfreeco/swift-dependencies

> Dependency injection for Swift modeled on SwiftUI's `Environment`, propagated through task-local values.

[GitHub repo](https://github.com/pointfreeco/swift-dependencies) ·
[Official website](https://www.pointfree.co) ·
[License: MIT](https://github.com/pointfreeco/swift-dependencies/blob/main/LICENSE)

## Overview

swift-dependencies is a dependency-management library from Point-Free (Brandon Williams and Stephen Celis), the Swift education company behind The Composable Architecture[^1]. It was extracted from TCA in 2022 so that dependency control could be used on its own, without adopting a whole architecture[^2]. The core idea mirrors SwiftUI's `@Environment`: dependencies live in a shared `DependencyValues` bag, are read through the `@Dependency` property wrapper, and are overridden for a scope rather than passed explicitly through initializers.

The library's real subject is not "injection" but *control*. Its argument is that `Date()`, `UUID()`, `Task.sleep`, clocks, file access, and network clients are all dependencies on the outside world, and that reaching for them directly makes code non-deterministic and hard to test or preview. By routing those calls through `@Dependency(\.date)`, `@Dependency(\.uuid)`, and friends, a feature can be run against a controlled clock, a fixed date, or an auto-incrementing UUID in tests and Xcode previews while still using live implementations in production[^3].

The defining tradeoff is the propagation mechanism. Overrides are carried by Swift's `@TaskLocal` storage, which means dependencies flow automatically down structured-concurrency call trees but silently *stop* at unstructured boundaries — a bare `Task { }`, an escaping completion handler, a `DispatchQueue.async`. This buys ergonomic, thread-safe, scoped overrides at the cost of a class of subtle bugs where a test override "doesn't take" because control escaped the task-local scope. Understanding that boundary is the price of admission.

## Getting Started

Add via Swift Package Manager:

```swift
dependencies: [
  .package(url: "https://github.com/pointfreeco/swift-dependencies", from: "1.0.0")
]
// then, per target:
.product(name: "Dependencies", package: "swift-dependencies"),
```

Declare dependencies on a model and use them instead of calling the global APIs directly:

```swift
import Dependencies

@Observable
final class FeatureModel {
  var items: [Item] = []

  @ObservationIgnored @Dependency(\.date.now) var now
  @ObservationIgnored @Dependency(\.uuid) var uuid
  @ObservationIgnored @Dependency(\.continuousClock) var clock

  func addButtonTapped() async throws {
    try await clock.sleep(for: .seconds(1))     // not Task.sleep
    items.append(Item(id: uuid(), createdAt: now))  // not UUID()/Date()
  }
}
```

Override for a single test with the `DependenciesTestSupport` trait:

```swift
import DependenciesTestSupport, Testing

@Test(.dependencies {
  $0.clock = .immediate
  $0.uuid = .incrementing
  $0.date.now = Date(timeIntervalSinceReferenceDate: 1234567890)
})
func add() async throws {
  let model = FeatureModel()
  try await model.addButtonTapped()
  #expect(model.items.first?.id == UUID(0))
}
```

## Architecture / How It Works

The library has three moving parts:

1. **`DependencyKey`** — you register a dependency by conforming a type to `DependencyKey` and providing a `liveValue`. Optionally you supply a `testValue` and `previewValue`. The key's associated `Value` is what `@Dependency` returns.
2. **`DependencyValues`** — a struct exposed through computed properties (like `var uuid: UUIDGenerator`) that reads and writes the current key values. Extending it with your own computed property is how a dependency becomes addressable via a key path such as `\.apiClient`.
3. **`@Dependency`** — a property wrapper that resolves a value out of the *current* `DependencyValues`, which is stored in a `@TaskLocal`.

Overrides happen through `withDependencies(_:operation:)`, which mutates a copy of the current values and installs it as the task-local for the duration of `operation`. Because it is task-local, the override is scoped, inherited by child tasks, and thread-safe by construction. `prepareDependencies` sets global defaults once (typically at app launch or in a preview) rather than for a scope.

The most important design decision is **context**. Code runs in one of three contexts — `live`, `preview`, or `test` — and `@Dependency` picks `liveValue`, `previewValue`, or `testValue` accordingly. Crucially, in the `test` context, accessing a dependency whose `testValue` was not provided (and left as the library's "unimplemented" default) triggers an immediate test failure[^3]. This turns "I forgot to mock something" from a silent interaction with the real world into a loud, actionable failure — the library's most distinctive safety property, and the same guarantee TCA relies on.

swift-dependencies is the dependency layer beneath The Composable Architecture; the two share this model, and swift-clocks (`ImmediateClock`, `TestClock`) and swift-concurrency-extras are companion packages that supply controllable versions of common system types[^2].

## Production Notes

**Task-local propagation is the recurring footgun.** Overrides do not cross into unstructured work. If a feature spins up a `Task { }`, stores an escaping closure, or hops onto `DispatchQueue`, `@Dependency` inside that work resolves against whatever the task-local was *there* — often the live/default values, not your override. The library provides `withEscapedDependencies` to capture the current context and re-enter it later, but you have to reach for it deliberately. Symptom: a test override that appears ignored, or a preview that unexpectedly hits the network.

**Test failures on unimplemented dependencies are intentional, not a bug.** New adopters frequently file "my test crashes/fails as soon as it touches a dependency." That is the design: every dependency a feature uses in a test must be explicitly overridden, or the test fails. It forces exhaustive mocking. Provide a `testValue` (often `unimplemented(...)` from swift-concurrency-extras) so the failure names exactly which dependency leaked.

**`prepareDependencies` vs `withDependencies`.** `prepareDependencies` can only be called before any dependency has been accessed and sets a global baseline; calling it too late (after first access) is itself a runtime warning. Use it once at the app entry point; use `withDependencies` everywhere else.

**Toolchain coupling.** The ergonomic test trait (`.dependencies { }`) lives in `DependenciesTestSupport` and targets Swift Testing; XCTest users use `withDependencies` directly. Newer features assume recent Swift toolchains and the Observation framework — pin versions if you support older Xcode. The 1.x line has held API stability, so upgrades within 1.x are generally low-risk[^4].

**Global-ish, but not a singleton.** Because state lives in task-locals, there is no shared mutable container to leak between tests running in parallel — a genuine advantage over service-locator containers. The cost is that reasoning about "what value will I get here" requires knowing the task tree, which is less obvious than reading a constructor argument.

## When to Use / When Not

**Use when:**
- You want deterministic tests and working Xcode previews for code that touches dates, UUIDs, clocks, network, or the file system.
- You already use, or plan to use, The Composable Architecture — this is its native dependency layer.
- You value scoped, parallel-test-safe overrides and the "unimplemented dependency fails the test" guarantee.

**Avoid when:**
- You want plain constructor injection with no property-wrapper magic and no task-local semantics — the indirection will feel like overhead.
- Your codebase is heavy on unstructured concurrency and legacy callback APIs, where the task-local boundary will bite repeatedly.
- You need compile-time-verified dependency graphs (missing-dependency errors at build time rather than runtime).

## Alternatives

- hmlongco/Factory — lightweight container-based DI; use when you want simple registration and resolution without task-local propagation.
- uber/needle — codegen-based, compile-time-safe DI; use when a large app needs missing dependencies caught at build time.
- Swinject/Swinject — classic runtime DI container; use when you want a traditional service-locator with object-graph scoping.
- scribd/Weaver — annotation + codegen DI; use when you prefer generated boilerplate over runtime resolution.
- pointfreeco/swift-composable-architecture — use when you want the full architecture, not just its dependency layer (superset of this library).

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2022-09 | Extracted from The Composable Architecture as a standalone library[^2]. |
| 1.0 | 2023 | First stable release; committed `@Dependency` / `DependencyValues` API[^4]. |
| 1.x | 2023–2026 | Ongoing: Swift Testing support via `DependenciesTestSupport`, Observation integration, refinements. |

## References

[^1]: Point-Free — video series on functional programming in Swift by Brandon Williams and Stephen Celis. https://www.pointfree.co
[^2]: swift-dependencies README, "Overview" and "Extensions" — origin as a component of The Composable Architecture. https://github.com/pointfreeco/swift-dependencies
[^3]: Dependencies documentation, "Live, preview, and test dependencies" and "Testing." https://swiftpackageindex.com/pointfreeco/swift-dependencies/main/documentation/dependencies
[^4]: Releases, pointfreeco/swift-dependencies. https://github.com/pointfreeco/swift-dependencies/releases

## Tags

swift, swiftui, dependency-injection, dependency-management, testing, xcode-previews, task-local, point-free, composable-architecture, ios
