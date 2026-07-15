# siteline/swiftui-introspect

> Reach the underlying UIKit/AppKit view behind a SwiftUI view when SwiftUI itself won't expose the knob you need.

[GitHub repo](https://github.com/siteline/swiftui-introspect) ·
[License: MIT](https://github.com/siteline/swiftui-introspect/blob/main/LICENSE)

## Overview

SwiftUI Introspect is an escape hatch. SwiftUI renders most of its controls on top of UIKit (iOS/tvOS/visionOS) or AppKit (macOS), but deliberately hides those backing objects. When you need something SwiftUI has no modifier for — disabling `UIScrollView` bounce, recoloring a `UINavigationBar`, reaching a `UITextField`'s delegate — this library hands you the real object so you can mutate it directly.

It works by planting two invisible marker views around the target and walking the UIKit/AppKit hierarchy between them to find the backing view, rather than reading private SwiftUI internals[^1]. That design choice is the whole story of the project: it uses only public APIs, makes no forced casts, and silently does nothing when the expected view can't be found — so a bad introspection degrades to a no-op instead of a crash. The maintainers explicitly call it production-suitable for this reason[^1].

The defining tension is version brittleness, and the library's answer to it is unusually honest. Because the private view hierarchy SwiftUI produces can change shape between OS major versions, every `.introspect` call requires you to *name the OS versions you've tested against* (`.iOS(.v17, .v18, ...)`). This is friction by design: it forces you to re-verify on each new OS rather than let a silent layout change break your app in the wild. The cost is that your introspection code needs a line edit every September when Apple ships a new iOS.

## Getting Started

Add it via Swift Package Manager:

```swift
// Package.swift
.package(url: "https://github.com/siteline/swiftui-introspect", from: "26.0.0"),
// target dependency:
.product(name: "SwiftUIIntrospect", package: "swiftui-introspect"),
```

```swift
import SwiftUI
import SwiftUIIntrospect

struct ContentView: View {
    var body: some View {
        ScrollView {
            Text("Item 1")
        }
        .introspect(.scrollView, on: .iOS(.v17, .v18)) { scrollView in
            scrollView.bounces = false          // real UIScrollView
        }
    }
}
```

The signature is `.introspect(_ viewType:, on platforms:, scope:, customize:)`. By default the modifier acts on its receiver; pass `scope: .ancestor` to reach an enclosing view instead.

## Architecture / How It Works

There is no reflection into SwiftUI's private types. The mechanics are:

1. `.introspect` inserts an invisible `IntrospectionView` (a representable) just before the target and an invisible anchor just after it.
2. At layout time it traverses the sibling/subview range between those two markers in the UIKit/AppKit tree, looking for the first instance of the expected class (e.g. `UIScrollView`).
3. If found, your closure runs with that instance; if not, nothing happens.

View support is modeled as types conforming to `IntrospectableViewType` (`.scrollView`, `.list`, `.textField`, …), each paired with per-platform, per-version `ViewVersion` descriptors that map a SwiftUI type to a concrete UIKit/AppKit class for a given OS. This is why the version list is mandatory: the descriptor for `.list` on iOS 15 resolves to a `UITableView`, but on iOS 16+ it resolves to a `UICollectionView` — a real backing-class change the library encodes explicitly rather than guessing.

Two things live behind an `@_spi(Advanced)` import for people who accept more risk: range-based version predicates (`.iOS(.v13...)`) that opt into *future* OS versions you haven't tested, and a `@Weak` property wrapper for holding an introspected instance beyond the closure without creating a retain cycle. The `@_spi` gate is deliberate — it keeps the sharp tools out of the default surface.

## Production Notes

- **The closure can run more than once.** It fires on view updates and re-renders, so it must be idempotent. Setting a property repeatedly is fine; appending to something or allocating is not.
- **Never mutate SwiftUI state inside the closure.** Doing so mid-layout invites "modifying state during view update" warnings and loops. If you must, wrap the write in `DispatchQueue.main.async`.
- **The mandatory version list is your upgrade tax.** When a new iOS/macOS ships you either add the new `.vXX` case (after verifying the backing class is unchanged) or your introspection quietly stops firing. Silent no-op is safer than a crash but easy to miss in QA — features "just stop working" on the new OS.
- **`scope: .ancestor` and default scope surprise people.** Calling `.introspect` from inside the view you want to introspect (rather than on it) does nothing; this is the most common "why isn't my closure called" report.
- **Retain cycles.** Capturing `self` strongly in the closure, or storing the instance in `@State`, can leak. Use `[weak self]` or the Advanced `@Weak` wrapper.
- **Some views can never be introspected.** `Text`, `Image`, stacks, `Spacer`, `Divider`, `Menu`, and `Chart` have no backing UIKit/AppKit view (or aren't the class you'd expect), and the docs list these as permanently unsupported[^1]. Reaching for introspection there is a dead end regardless of OS version.

## When to Use / When Not

**Use when:**
- SwiftUI genuinely lacks the API and the fix lives on a UIKit/AppKit property (scroll bounce, keyboard/return-key traits, nav bar chrome, cell separators).
- You control the app and can commit to re-testing each `.introspect` on new OS releases.
- You want a targeted patch, not a re-implementation of the whole control.

**Avoid when:**
- A native SwiftUI modifier exists — always prefer it; introspection is the last resort.
- You need `Text`, `Image`, stacks, `Menu`, or `Chart` internals — unsupported by construction.
- You want set-and-forget code that survives OS upgrades untouched; the version-pinning model is the opposite of that.

## Alternatives

- SwiftUIX/SwiftUIX — a broad add-on library that reimplements or wraps many UIKit-backed components natively; reach for it when SwiftUI lacks a *component entirely* rather than when you need to tweak an existing one.
- Apple's `UIViewRepresentable` / `UIViewControllerRepresentable` (native) — build the UIKit view yourself when you own the control outright; more code, zero introspection fragility.
- Native SwiftUI modifiers — for anything Apple has since exposed (many scroll/list/toolbar knobs landed in iOS 16–17), the first-party modifier is the correct answer and retires the introspection hack.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-11 | First release as "Introspect for SwiftUI"; old `.introspectScrollView { }` per-control modifier API[^2]. |
| 1.0.0 | 2023 | Full rewrite: unified `.introspect(_:on:)` API, explicit per-version targeting, `IntrospectableViewType`, `@_spi(Advanced)` surface[^3]. |
| 26.x / 27.0.0-beta | 2025–2026 | Later majors track platform support; version descriptors extended through iOS/tvOS 26–27 and matching macOS/visionOS[^4]. |

The project describes itself as effectively feature-complete: no new capabilities are planned, only new platform versions and view types as Apple ships them[^1].

## References

[^1]: SwiftUI Introspect README — "How it works", "Usage in production", "General Guidelines", "Cannot implement", "Note for library authors". https://github.com/siteline/swiftui-introspect
[^2]: Repository created 2019-11-26; original per-control modifier API predates the 1.0 rewrite. https://github.com/siteline/swiftui-introspect
[^3]: 1.0 rewrite introduced the `IntrospectableViewType` + version-descriptor model shown in the current README's advanced-usage section. https://github.com/siteline/swiftui-introspect#implement-your-own-introspectable-type
[^4]: Current install snippet targets `from: "27.0.0-beta"` and version cases include `.v26`/`.v27`. https://swiftpackageindex.com/siteline/swiftui-introspect

## Tags

swift, swiftui, uikit, appkit, ios, macos, introspection, view-hierarchy, apple, spm
</content>
</invoke>
