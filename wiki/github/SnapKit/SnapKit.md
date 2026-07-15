# SnapKit/SnapKit

> A Swift DSL over UIKit/AppKit Auto Layout — constraints as chained method calls instead of `NSLayoutConstraint` boilerplate.

[GitHub repo](https://github.com/SnapKit/SnapKit) ·
[Official website](https://snapkit.github.io/SnapKit/) ·
[License: MIT](https://github.com/SnapKit/SnapKit/blob/develop/LICENSE)

## Overview

SnapKit is a thin domain-specific language that sits on top of Apple's Auto Layout. It does not implement a layout engine of its own; it constructs and activates ordinary `NSLayoutConstraint` objects, but exposes them through a fluent, chainable API (`make.top.equalTo(view).offset(10)`) instead of the verbose constraint constructors or the fragile point-and-click of Interface Builder. The repository was created in June 2014 — the same month Swift was announced — which makes it one of the oldest Swift libraries still in wide use[^1]. It began life under the name "Snappy" and is the Swift successor to Masonry, the same author's Objective-C constraint DSL.

The audience is any team writing UIKit or AppKit (and tvOS) layouts in code. SnapKit's value is entirely ergonomic: fewer lines, readable constraint intent, and edits that live in source control rather than a binary `.xib`. As of 2026 it has roughly 20k stars and remains actively maintained — the `6.x` line was retargeted to Swift 6, Xcode 26, and iOS 14+ / macOS 12+ / tvOS 14+[^2].

The defining tension is generational. SnapKit is excellent at the problem it solves, but that problem — imperative Auto Layout — is the one Apple is steering developers away from with SwiftUI's declarative layout. SnapKit has no SwiftUI story and is not meant to; it is infrastructure for the large body of existing and new-but-UIKit codebases, not for greenfield declarative apps.

## Getting Started

Swift Package Manager (the manual/CocoaPods paths still exist but SPM is the default):

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/SnapKit/SnapKit.git", .upToNextMajor(from: "6.0.0"))
]
```

```swift
import SnapKit

final class MyViewController: UIViewController {
    let box = UIView()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(box)                 // MUST be in the hierarchy first
        box.snp.makeConstraints { make in
            make.width.height.equalTo(50)
            make.center.equalToSuperview()
        }
    }
}
```

The whole API hangs off the `snp` namespace added to every view. `makeConstraints` is the common entry point; `updateConstraints` and `remakeConstraints` handle mutation (see Production Notes — the distinction is a frequent footgun).

## Architecture / How It Works

Every layout-capable view gains a `snp` property (a `ConstraintViewDSL`). Inside a `makeConstraints` closure the `make` argument is a `ConstraintMaker`. Chained keywords (`top`, `leading`, `width`, `centerX`, …) accumulate into a `ConstraintDescription` describing attributes as a bitmask (`ConstraintAttributes`). Relational calls (`equalTo`, `lessThanOrEqualTo`, `greaterThanOrEqualTo`) plus modifiers (`.offset()`, `.inset()`, `.multipliedBy()`, `.priority()`) finalize each description. When the closure returns, SnapKit resolves those descriptions into concrete `NSLayoutConstraint` instances and activates them.

There is no runtime magic beyond that construction step. SnapKit is a builder for the same constraint objects you would write by hand, which means everything the underlying Cassowary-based Auto Layout solver does — the actual layout math, constraint priorities, unsatisfiable-constraint logging — happens exactly as it would without SnapKit. The library's surface area is small and stable; most releases are compiler/toolchain retargets rather than feature work.

Coupling is straight to UIKit/AppKit. SnapKit imports `UIKit` on iOS/tvOS and `AppKit` on macOS and operates on `UIView`/`NSView`, layout guides, and `UILayoutSupport`. It has no third-party dependencies. It has no relationship to SwiftUI's layout system, Core Animation layout, or manual `frame` math — those are separate worlds it does not touch.

## Production Notes

- **`make` vs `update` vs `remake` is the classic bug.** Calling `makeConstraints` a second time on the same view *adds* a new set of constraints on top of the old ones, producing conflicts and console warnings. Use `updateConstraints` to change only the constants of already-installed constraints (it cannot add or change relationships), and `remakeConstraints` to tear down all SnapKit-made constraints on that view and rebuild from scratch. Reaching for `make` when you meant `remake` is a routine source of "Unable to simultaneously satisfy constraints" spam.
- **View must be in the hierarchy first.** Calling `makeConstraints` before `addSubview` (or referencing a view with no common ancestor) crashes or silently fails to install. Order matters.
- **No compile-time safety.** Constraint conflicts, ambiguous layouts, and priority mistakes surface only at runtime as Auto Layout console logs — SnapKit's fluent syntax does not catch them earlier. Ambiguity debugging is still done with `hasAmbiguousLayout` / `exerciseAmbiguityInLayout` and the view-debugger, same as raw Auto Layout.
- **Performance is Auto Layout's, not SnapKit's.** The DSL overhead is negligible, but the Cassowary solver scales poorly with deep or densely inter-constrained view trees. For large scrolling lists or hot re-layout paths, the bottleneck is Auto Layout itself; teams sometimes drop to manual `layoutSubviews`, or move perf-critical screens to frame math or SwiftUI, regardless of SnapKit.
- **Main thread only.** Constraint mutation must happen on the main thread; there is no async layout.
- **Toolchain-gated upgrades.** Major SnapKit versions track Swift/Xcode, not features. The `6.x` line requires Swift 6 and Xcode 26 and drops older OS targets; pinning `.upToNextMajor` avoids being dragged onto a new Xcode before you are ready[^2]. The Swift 5 requirement note in the README applies to the `5.x` line.

## When to Use / When Not

**Use when:**
- You are building UIKit/AppKit/tvOS UI in code and want readable constraints without `NSLayoutConstraint` verbosity or `.xib` merge pain.
- Your team already thinks in Auto Layout and wants ergonomics, not a new mental model.
- You want a small, dependency-free, long-stable library that mostly just tracks the compiler.

**Avoid when:**
- You are starting fresh and can adopt SwiftUI — its declarative layout is the native path forward and SnapKit offers nothing there.
- You want zero dependencies: Apple's `NSLayoutAnchor` API has been reasonable since iOS 9 and covers many cases.
- The screen is performance-critical with large/dynamic constraint graphs, where any Auto Layout approach will be the bottleneck.

## Alternatives

- Apple NSLayoutAnchor (built into UIKit/AppKit) — use when you want no dependency and the native anchor API's slightly heavier syntax is acceptable.
- roberthein/TinyConstraints — use when you want an even terser, more implicit syntax than SnapKit's explicit chains.
- PureLayout/PureLayout — use in Objective-C or mixed codebases wanting a category-based constraint API.
- SnapKit/Masonry — use for pure Objective-C projects; it is this project's Objective-C predecessor.
- Apple SwiftUI — use when you can adopt declarative UI and leave imperative Auto Layout behind entirely.

## History

| Version | Date | Notes |
|---------|------|-------|
| Snappy 0.1 | 2014 | Initial release as "Snappy," days after Swift's announcement[^1]. |
| 0.x → 1.0 | 2015 | Renamed to SnapKit to avoid naming collisions. |
| 3.0 | 2016 | Swift 3 language migration. |
| 4.0 | 2017 | Swift 4 support. |
| 5.0 | 2019 | Swift 5 support (README requires >= 5.0.0 for Swift 5.x)[^3]. |
| 6.0 | 2025–2026 | Swift 6 / Xcode 26, iOS 14+ / macOS 12+ / tvOS 14+ targets[^2]. |

## References

[^1]: Repository metadata — created 2014-06-05, MIT license, ~20.3k stars / ~2.0k forks as of 2026-07. https://github.com/SnapKit/SnapKit
[^2]: SnapKit README — Requirements: iOS 14.0+ / macOS 12.0+ / tvOS 14.0+, Xcode 26.0+, Swift 6.0+; SPM `.upToNextMajor(from: "6.0.0")`. https://github.com/SnapKit/SnapKit#requirements
[^3]: SnapKit README migration note — "To use with Swift 5.x please ensure you are using >= 5.0.0." https://github.com/SnapKit/SnapKit
[^4]: SnapKit documentation and F.A.Q. https://snapkit.github.io/SnapKit/docs/

## Tags

swift, ios, macos, tvos, auto-layout, uikit, appkit, dsl, constraints, ui-layout, cocoapods
