# Quick/Nimble

> Matcher-style assertions (`expect(x).to(equal(y))`) for Swift and Objective-C tests, with first-class support for polling and async expectations.

[GitHub repo](https://github.com/Quick/Nimble) ·
[Documentation](https://quick.github.io/Nimble/documentation/nimble/) ·
[License: Apache-2.0](https://github.com/Quick/Nimble/blob/main/LICENSE)

## Overview

Nimble is an assertion library for test code. Instead of `XCTAssertEqual(a, b)`, you write `expect(a).to(equal(b))` — a fluent matcher API inspired by Rspec's `expect`, Cedar, and the wider BDD lineage[^1]. It ships as part of the Quick organization but is deliberately independent of Quick itself: Nimble is the assertions, Quick is the `describe`/`it` spec runner, and either can be used without the other. Most teams that adopt Nimble use it directly on top of XCTest, not Quick.

The library's stated reason to exist is failure-message quality. When `expect(["Atlantic", "Pacific"]).to(contain("Mississippi"))` fails, the message names the collection, the expected element, and the operation in plain English — richer than the bare "XCTAssertEqual failed" you get from the standard framework. That, plus operator overloads (`expect(3) > 2`) and the async polling matchers, is what it trades a third-party dependency for.

The defining tension is exactly that dependency. Nimble is a testing-only convenience layer on a first-party foundation (XCTest, and now Apple's swift-testing) that keeps improving underneath it. Adopting it means every test target carries an external framework, some matchers (`throwAssertion`, `raiseException`) depend on Objective-C exception-catching machinery that is not available on every install path, and the payoff is ergonomics rather than capability. Whether that trade is worth it is a team-by-team call, and increasingly one made against Apple's own `#expect` macro.

## Getting Started

Swift Package Manager (add to a **test** target, never an app target):

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/Quick/Nimble.git", from: "13.0.0"),
],
targets: [
    .testTarget(
        name: "MyLibraryTests",
        dependencies: ["MyLibrary", "Nimble"]),
]
```

```swift
import Nimble
import XCTest

final class OceanTests: XCTestCase {
    func testExpectations() {
        expect(1 + 1).to(equal(2))
        expect(1.2).to(beCloseTo(1.1, within: 0.1))
        expect("seahorse").to(contain("sea"))
        expect(["Atlantic", "Pacific"]).toNot(contain("Mississippi"))

        // Polls the value until it passes or the timeout elapses
        expect(ocean.isClean).toEventually(beTruthy())
    }
}
```

Also installable via CocoaPods, Carthage, or git submodule (see the README). Note: installing through SPM drops the `raiseException` matcher, which relies on Objective-C exception interop not linked on that path[^2].

## Architecture / How It Works

The core is one generic entry point, `expect(...)`, which wraps a value (or an autoclosure expression) in an `Expectation`. You then apply a **matcher** — a small object holding predicate logic plus a description used to render pass/fail messages. `to`, `toNot`/`notTo`, and the polling variants drive the matcher against the captured expression and report to the underlying assertion handler (XCTest by default).

Matchers are the extension surface. Historically the type was named `Predicate`; Nimble 13 renamed it to `Matcher` and deprecated the old name, a rename that touched every custom-matcher codebase in the ecosystem[^3]. A matcher is a closure from an expression to a `MatcherResult` (status + failure message), so writing your own is a few lines and does not require subclassing.

Async and polling are the interesting internals:

- **`toEventually` / `toEventuallyNot` / `toAlways` / `toNever`** re-evaluate the expression on an interval until it passes (or fails, for the negative forms) or a timeout is hit. Under the hood this spins the run loop, which is why these matchers interact subtly with main-thread and main-actor code.
- **Swift Concurrency support** (added in the v12 line) lets you `await expect(await something()).to(...)` and integrates the polling matchers with the concurrency runtime rather than only the run loop[^4]. `AsyncDefaults` / `PollingDefaults` configure the global timeout and poll interval.

Objective-C is a first-class target, not an afterthought: `expect(@1).to(equal(@1))` works from `.m` files via the `NMB*` bridging types, and much of the exception-based matcher behavior exists to serve that side.

Nimble does not run tests, discover them, or manage lifecycle. It is a leaf dependency that plugs into whatever runner is present. That narrow scope is why it composes cleanly with XCTest, with Quick, and — with community shims — with newer runners.

## Production Notes

"Production" here means CI test suites, not shipped binaries. Nimble must never be linked into an App Store build; it is test infrastructure only, and the maintainers state it collects no analytics[^5].

- **Polling matchers are the main footgun.** `toEventually` spins the run loop while polling. On the main actor / main thread this can deadlock or fail to observe updates if the code under test needs the same thread to make progress. Prefer the async (`await`) forms for `async` code, and keep default timeouts realistic — a too-short `PollingDefaults.timeout` produces flaky failures under CI load, a too-long one hides real hangs.
- **Install-path matters for exception matchers.** `raiseException` (Obj-C) is unavailable on SPM installs, and `throwAssertion` relies on precondition-catching support that is platform-limited. Code that asserts on fatal errors or Obj-C exceptions is not portable across every install method or platform.
- **Version churn tracks Xcode/Swift.** Nimble follows the current Swift toolchain closely; major versions have raised the minimum Swift/Xcode and dropped older platform support. Pin the version and expect a real (if usually mechanical) migration on major bumps.
- **The `Predicate` → `Matcher` rename** (v13) is the most disruptive recent change for anyone with custom matchers: the old API is deprecated but the type, its result type, and helper names all shifted. Budget a find-and-replace pass when upgrading across that boundary[^3].
- **Failure-message quality is the actual product.** If your team does not read matcher output (e.g. CI-only, logs ignored), most of Nimble's value over raw `XCTAssert` evaporates and the dependency is harder to justify.

## When to Use / When Not

**Use when:**
- You want readable, self-describing assertions and value the diagnostic messages on failure.
- Your suite is heavy on asynchronous or eventually-consistent state and you want `toEventually` / `await expect` polling rather than hand-rolled expectations.
- You already use Quick, or you are porting an Rspec/Cedar-style suite and want familiar `expect` ergonomics.
- You test mixed Swift + Objective-C code and want one matcher API across both.

**Avoid when:**
- You are starting fresh and can adopt Apple's swift-testing, whose built-in `#expect`/`#require` macros cover much of the same ground with no third-party dependency.
- You want zero test-target dependencies and `XCTAssert*` is good enough.
- Your matchers would be exception/precondition-based on install paths (SPM) or platforms where that support is absent.

## Alternatives

- apple/swift-testing — Apple's first-party framework; use instead when starting new suites and you want `#expect` macros without an external dependency.
- Quick/Quick — the sister BDD spec runner; not a competitor to the matchers but the thing Nimble is most often paired with, and enough on its own if you only want `describe`/`it` structure.
- XCTest (Apple, bundled) — the `XCTAssert*` baseline; use instead when you want no dependency and don't need rich failure messages or polling.
- pointfreeco/swift-snapshot-testing — different niche (reference snapshots); use instead when you assert on rendered output or large value trees rather than scalar expectations.
- pointfreeco/swift-custom-dump — pretty diffing/`XCTAssertNoDifference`; use instead when your pain is specifically unreadable equality diffs, not assertion syntax.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-06 | Repository created; matcher framework inspired by Cedar[^1]. |
| 12.x | 2023 | Swift Concurrency support — `await expect`, async polling matchers[^4]. |
| 13.0 | 2023–2024 | `Predicate` renamed to `Matcher`; minimum Swift/Xcode raised[^3]. |
| 13.2 | 2024 | Referenced as the current Carthage line in the README. |

(Exact release dates for individual tags are in the repository's CHANGELOG; only milestones verified from the README and changelog are listed above.)

## References

[^1]: Nimble README — "Inspired by Cedar." https://github.com/Quick/Nimble
[^2]: Nimble README, Swift Package Manager section — "if you install Nimble using Swift Package Manager, then `raiseException` is not available." https://github.com/Quick/Nimble#swift-package-manager
[^3]: Nimble CHANGELOG — `Predicate` renamed to `Matcher` (v13 line). https://github.com/Quick/Nimble/blob/main/CHANGELOG.md
[^4]: Nimble documentation — asynchronous expectations / polling. https://quick.github.io/Nimble/documentation/nimble/
[^5]: Nimble README, Privacy Statement — testing-only, no analytics or tracking. https://github.com/Quick/Nimble#privacy-statement

## Tags

swift, objective-c, testing, assertions, matchers, bdd, xctest, async-testing, ios, macos
