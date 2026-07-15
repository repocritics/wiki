# ReactiveX/RxSwift

> Reactive programming for Swift — the Swift port of the ReactiveX standard, now a mature library competing against Apple's own Combine and Swift Concurrency.

[GitHub repo](https://github.com/ReactiveX/RxSwift) ·
[ReactiveX.io](https://reactivex.io) ·
[License: MIT](https://github.com/ReactiveX/RxSwift/blob/main/LICENSE.md)

## Overview

RxSwift is the Swift-specific implementation of Reactive Extensions (Rx), the family of libraries that also includes RxJS, RxJava, and Rx.NET[^1]. Its central abstraction is `Observable<Element>`: a push-based stream that emits `next`, `error`, and `completed` events, which callers transform and compose with operators (`map`, `flatMap`, `filter`, `combineLatest`, `throttle`, and hundreds more). The pitch is that KVO, UI events, network calls, timers, and notifications all become the same kind of sequence, so asynchronous glue code collapses into declarative pipelines. It was created by Krunoslav Zaher in 2015 and is now maintained primarily by Shai Mishali[^2].

The defining tension in 2026 is that RxSwift solves a problem Apple has since addressed in-house. Combine (2019, iOS 13+) covers most of the same ground as a first-party framework, and Swift Concurrency (`async`/`await`, `AsyncSequence`, actors) plus the Observation macro cover much of the rest without a dependency[^3]. RxSwift's remaining advantages are its enormous operator catalog, cross-platform reach (including Linux), support for OS versions below Combine's iOS 13 floor, and a decade of battle-tested MVVM patterns and community extensions. Its cost is a steep learning curve, a large binary, and the well-known memory and threading footguns that come with a manual reactive runtime.

## Getting Started

```swift
// Package.swift (Swift Package Manager)
dependencies: [
  .package(url: "https://github.com/ReactiveX/RxSwift.git", .upToNextMajor(from: "6.0.0"))
]
// link RxSwift and, for UI, RxCocoa (product of the same package)
```

```swift
import RxSwift
import RxCocoa

let disposeBag = DisposeBag()

// A search field bound reactively to a table view
searchBar.rx.text.orEmpty
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .flatMapLatest { query -> Observable<[Repository]> in
        query.isEmpty ? .just([]) : searchGitHub(query).catchAndReturn([])
    }
    .observe(on: MainScheduler.instance)
    .bind(to: tableView.rx.items(cellIdentifier: "Cell")) { _, repo, cell in
        cell.textLabel?.text = repo.name
    }
    .disposed(by: disposeBag)   // subscription lifetime tied to the bag
```

## Architecture / How It Works

RxSwift ships as five interdependent modules, all in one package[^4]:

- **RxSwift** — the core. `Observable`, `Observer`, `Disposable`, `Scheduler`, `Subject`, and the operator library. No dependencies.
- **RxCocoa** — Cocoa bindings: `.rx` reactive extensions on UIKit/AppKit, `Driver`, `Signal`, `ControlProperty`, KVO bridging. Depends on RxSwift and RxRelay.
- **RxRelay** — `PublishRelay`, `BehaviorRelay`, `ReplayRelay`: Subjects that never emit `error`/`completed`, used as UI-safe state holders.
- **RxTest** / **RxBlocking** — testing support (virtual-time `TestScheduler`, marble tests, synchronous waiting).

The runtime model is: an `Observable` is a cold description of work; subscribing runs it and returns a `Disposable`. Disposal must be managed manually, usually by adding subscriptions to a `DisposeBag` whose deinit tears them down. Threading is explicit and split into two operators that everyone eventually confuses: `subscribe(on:)` controls where the subscription work starts, `observe(on:)` controls where downstream events are delivered.

Above raw Observables sit **Traits** — constrained wrappers encoding intent: `Single` (one value or error), `Maybe`, `Completable`, and the RxCocoa `Driver`/`Signal` which guarantee main-thread delivery and never error out, making them the idiomatic type for binding to UI. Much of "learning RxSwift" is learning which trait to reach for and how to convert between them.

## Production Notes

- **Retain cycles are the number-one bug.** Capturing `self` strongly inside operator closures keeps the owner alive as long as the subscription lives. The `[weak self]` / `[unowned self]` discipline is mandatory, and RxCocoa's `weakify`/`withUnretained` helpers exist precisely because teams got this wrong repeatedly.
- **DisposeBag lifetime is the leak surface.** A subscription outlives its intended scope whenever the bag is owned by the wrong object or a long-lived Subject holds a reference. Leaks here are silent — no crash, just growing memory and duplicated side effects.
- **Threading mistakes surface as UI-thread crashes.** Forgetting `observe(on: MainScheduler.instance)` before a UI bind produces "must be used from main thread only" faults that only appear under specific timing.
- **Swift Package Manager has a long-standing cross-dependency bug (SR-12303)** that can break builds when multiple Rx modules are pulled transitively; the README still flags it, and CocoaPods or Carthage remain the more reliable integration paths for complex graphs[^5].
- **Debugging is hard.** Stack traces run through the operator machinery, not your code. The `.debug()` operator and disciplined naming are the practical tools; expect to read marble diagrams to reason about `flatMap` vs `flatMapLatest` vs `concatMap`.
- **Binary size and compile time.** The operator catalog is large and heavily generic, which inflates build times and app size versus a Combine or async/await equivalent.
- **Migration inertia.** Large RxSwift codebases are expensive to move off; the common 2026 pattern is freezing RxSwift for existing screens while writing new code in Combine or Swift Concurrency, bridging at the boundary.

## When to Use / When Not

**Use when:**
- You maintain an existing RxSwift/RxCocoa MVVM codebase and want consistency.
- You must support OS versions below iOS 13 / macOS 10.15, where Combine is unavailable.
- You need operators or cross-platform (Linux) behavior that Combine does not offer.
- Your team already knows Rx and values the mature ecosystem (RxDataSources, RxGesture, etc.).

**Avoid when:**
- You are starting fresh and can target iOS 13+ — Combine or Swift Concurrency avoid the third-party dependency.
- Your async needs are simple; `async`/`await` is far easier to read and debug for linear flows.
- Your team is new to reactive programming and the learning-curve and footgun cost is not justified.
- Binary size, compile time, or dependency minimalism are priorities.

## Alternatives

- apple/swift — Combine (bundled in the SDK): first-party reactive framework, iOS 13+; use it when you can drop older-OS support and want no dependency.
- apple/swift — Swift Concurrency (`async`/`await`, `AsyncSequence`): use for linear asynchronous flows where readability beats operator composition.
- ReactiveCocoa/ReactiveSwift: the other mature Swift FRP library; use when you prefer its signal/property model and API conventions.
- pointfreeco/swift-composable-architecture: use when you want an opinionated end-to-end state-management architecture rather than a raw stream primitive.
- apple/swift-async-algorithms: use when you want Rx-like combinators (debounce, merge, zip) built on `AsyncSequence` instead of Observables.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015 | Initial release by Krunoslav Zaher[^2]. |
| 2.0 | 2016 | Swift 2 support. |
| 3.0 | 2016-10 | Swift 3, large API renaming. |
| 4.0 | 2017-10 | Swift 4, Traits consolidation. |
| 5.0 | 2019-12 | Swift 5, `Result`-based APIs. |
| 6.0 | 2021-01 | RxRelay split out, XCFrameworks, `bind`/operator renames[^6]. |
| 6.10.0 | 2026 | Latest 6.x maintenance release[^7]. |

## References

[^1]: ReactiveX — the Rx family and standard. https://reactivex.io/
[^2]: RxSwift repository and maintainers. https://github.com/ReactiveX/RxSwift
[^3]: RxSwift's own comparison with Combine and ReactiveSwift. https://github.com/ReactiveX/RxSwift/blob/main/Documentation/ComparisonWithOtherLibraries.md
[^4]: RxSwift module structure (RxSwift / RxCocoa / RxRelay / RxTest / RxBlocking). https://github.com/ReactiveX/RxSwift#understand-the-structure
[^5]: Swift Package Manager cross-dependency bug SR-12303, flagged in the RxSwift README. https://github.com/ReactiveX/RxSwift/issues/2127
[^6]: RxSwift 6 release notes. https://github.com/ReactiveX/RxSwift/releases/tag/6.0.0
[^7]: RxSwift releases. https://github.com/ReactiveX/RxSwift/releases

## Tags

swift, ios, reactive-programming, reactivex, functional-reactive, observable, rxcocoa, mvvm, cocoa, asynchronous
