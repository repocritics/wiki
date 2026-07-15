# pointfreeco/swift-snapshot-testing

> Value-based snapshot testing for Swift — assert any value against a recorded reference, not just screenshots of views.

[GitHub repo](https://github.com/pointfreeco/swift-snapshot-testing) ·
[Point-Free episode](https://www.pointfree.co/episodes/ep41-a-tour-of-snapshot-testing) ·
[License: MIT](https://github.com/pointfreeco/swift-snapshot-testing/blob/main/LICENSE)

## Overview

SnapshotTesting is a Swift testing library from Point-Free (Brandon Williams and Stephen Celis) that records a serialized reference of a value on first run and, on every subsequent run, compares the live value against that reference and fails on any difference[^1]. It is the de facto snapshot testing library in the iOS/macOS community, and its distinguishing idea is generality: where most snapshot libraries snapshot a `UIImage` of a `UIView`, this one snapshots *any* value into *any* format — images, `recursiveDescription` text, JSON, property lists, raw `URLRequest` dumps, or a `Mirror`-based `dump` that works on any Swift value with zero configuration.

The design grew out of Point-Free's "witness-oriented" library-design series[^2]. A snapshot strategy is not a protocol you conform to — it is a value, `Snapshotting<Value, Format>`, that bundles a way to turn a `Value` into a diffable `Format` plus a way to diff and render that format. Strategies are therefore first-class: you can transform an existing one (`.image` → `.image(on: .iPhoneSe)`), compose them, or build your own from an image/string/data conversion. This is the library's central tension: the value-oriented API is elegant and endlessly extensible, but image snapshots — the most common use — inherit all the environmental fragility of pixel comparison, which no amount of API elegance removes.

The library predates and now spans two test runners: it works from `XCTest` and, since the Swift Testing integration, from `@Test`/`@Suite` as well[^3]. Recording behavior is controlled globally or per-scope rather than per-assertion boolean flags, which is cleaner but is itself a footgun (see Production Notes).

## Getting Started

Add the package to a **test** target (not the app target) via SwiftPM:

```swift
dependencies: [
  .package(url: "https://github.com/pointfreeco/swift-snapshot-testing", from: "1.12.0"),
],
targets: [
  .testTarget(name: "MyAppTests", dependencies: [
    .product(name: "SnapshotTesting", package: "swift-snapshot-testing"),
  ]),
]
```

```swift
import SnapshotTesting
import Testing

@MainActor
struct MyViewControllerTests {
  @Test func layout() {
    let vc = MyViewController()

    // Image of the rendered view, pinned to a device profile:
    assertSnapshot(of: vc, as: .image(on: .iPhone13))

    // Textual view hierarchy — diffs are human-readable in CI logs:
    assertSnapshot(of: vc, as: .recursiveDescription)
  }

  @Test func apiClient() {
    // Not just views — snapshot any value:
    assertSnapshot(of: makeSignupRequest(), as: .raw) // dumps method, headers, body
    assertSnapshot(of: user, as: .json)               // Encodable → pretty JSON
  }
}
```

The first run records a reference PNG/text file under `__Snapshots__/` next to the test and **fails deliberately**, printing the path. Commit that file; the next run compares against it.

## Architecture / How It Works

The core type is `Snapshotting<Value, Format>`: `snapshot: (Value) -> Async<Format>`, plus a `Diffing<Format>` witness that knows how to serialize the format to `Data`, deserialize it, and produce a diff (with an optional attachment for the Xcode report navigator). `assertSnapshot(of:as:)` is a thin driver: run the strategy, look for a reference file at a path derived from the test's file/function/line, and either record (no reference exists, or recording is on) or compare.

Because everything routes through `Diffing`, the platform-specific machinery is isolated. `.image` strategies live in the UIKit/AppKit/SwiftUI-facing code and render a view into a `CGContext`; the diff is a pixel comparison with configurable tolerance. `.recursiveDescription`, `.dump`, `.json`, `.plist`, and `.raw` are pure string/data strategies with no rendering dependency, which is why they run on Linux and in command-line Swift where there is no simulator.

Strategy transformation is the mechanism behind device-agnostic snapshots. `.image(on: .iPhoneSe)` overrides the trait collection and view size before rendering, so a single simulator can produce references for many device/orientation/size-class combinations. `pullback`/`asyncPullback` adapt a strategy for `Value` into one for a different type by supplying a conversion function — this is how new strategies are built without touching the core.

Recording is stateful, controlled by `withSnapshotTesting(record:)` (a scoped override), the `record:` argument on an assertion, or the `.snapshots(record:)` test trait. Modes are `.all`, `.failed`, `.missing`, and `.never`; the default records only when no reference exists. The global `diffTool`/`diffToolCommand` hook lets failure messages print a ready-to-run external diff command (e.g. Kaleidoscope's `ksdiff`).

## Production Notes

**Image snapshots are environment-bound — this is the dominant operational cost.** A reference PNG encodes the exact output of a specific Xcode version, simulator OS, device profile, and host architecture. Rendering differs across Xcode releases (font hinting, anti-aliasing, sub-pixel rounding), and historically differed between Intel and Apple Silicon hosts. The practical consequence: references recorded on a developer's machine routinely fail on CI, and an Xcode upgrade can invalidate an entire snapshot suite at once. The README warns explicitly that snapshots must be compared on the same simulator that recorded them.

**Precision knobs exist because exactness is unachievable.** Image strategies accept `precision` (fraction of pixels allowed to differ) and `perceptualPrecision` (per-pixel color tolerance). Teams almost always need to loosen these below 1.0 to survive cross-machine rendering drift — at the cost of masking real regressions. Pinning the CI Xcode/simulator version is the more reliable fix; precision tuning is the pragmatic one.

**`record: .all` left in a commit is a silent-pass footgun.** In recording mode the library overwrites the reference and passes, so a stray recording override (or a scoped `withSnapshotTesting(record: .all)`) makes every assertion in scope accept whatever the code currently produces. Regressions get baked into the references and go green. Review diffs for recording flags, and prefer `.missing`/`.failed` over `.all` in shared config.

**Reference files are binary and live in git.** Image-heavy suites add megabytes of PNGs to the repository and to every clone. Some teams use the HEIC plug-in to shrink them, or Git LFS. Text strategies (`.recursiveDescription`, `.json`, `.dump`) avoid this entirely and produce reviewable diffs in pull requests — a strong reason to prefer them where a textual representation captures what you care about.

**SwiftUI snapshotting is indirect.** A SwiftUI `View` is hosted in a `UIHostingController` and rendered; layout timing, safe-area insets, and dynamic type can make results depend on how the view is embedded. Expect to specify a device/size explicitly rather than relying on intrinsic sizing.

**Global mutable configuration.** `diffTool`, `isRecording`/record state, and related settings are process-global; `withSnapshotTesting` scopes them, but ad hoc mutation of the globals in parallelized tests can interfere. Prefer the scoped API and the `.snapshots` trait.

## When to Use / When Not

**Use when:**
- You want regression coverage for rendered UI (UIKit/AppKit/SwiftUI) and can pin a stable CI toolchain.
- You want to lock down *non-visual* output — API request shapes, `Codable` payloads, generated HTML, formatter output, any value with a stable textual form.
- You value low-ceremony tests: no manual reference construction, snapshots record themselves on first run.
- You're already in the Point-Free ecosystem (Composable Architecture, swift-custom-dump) and want consistent tooling.

**Avoid when:**
- Your CI can't guarantee a fixed Xcode/simulator, and you can't tolerate flaky image diffs — the maintenance tax on image snapshots will dominate.
- You need precise, assertable invariants ("this label equals X") — a regular `#expect`/`XCTAssertEqual` states intent better than a snapshot that passively captures everything.
- You're on a non-Apple platform and need view rendering — the image strategies depend on Apple UI frameworks (text/data strategies still work on Linux).
- Binary reference bloat in git is unacceptable and your outputs are inherently visual.

## Alternatives

- uber/ios-snapshot-test-case — the original ObjC FBSnapshotTestCase lineage; image-only, `UIView`-centric. Use it when you have a legacy suite already built on it, otherwise this library supersedes it.
- cashapp/paparazzi — JVM-side rendering of Android/Compose UI with no emulator. Use it when your UI is Android, where it removes the device-dependence problem entirely.
- jestjs/jest — the JavaScript snapshot testing that inspired this library's diffing and auto-record UX. Use it when your surface is a JS/TS codebase.
- pointfreeco/swift-custom-dump — same authors; value diffing without on-disk reference files. Use it when you want inline `expectNoDifference`-style assertions rather than recorded snapshots.
- doordash-oss/swiftui-preview-snapshots — builds on this library to share SwiftUI `Preview` configs with snapshot tests. Use it when you want previews and snapshots to stay in sync from one source.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2017-07-07 | Repository opened by Point-Free[^4]. |
| Ep. 41 | 2018 | "A Tour of Snapshot Testing" — public introduction of the library[^1]. |
| 1.0 era | 2018–2019 | Stable `assertSnapshot` API; strategy-value ("witness") design[^2]. |
| 1.x | 2022–2023 | `perceptualPrecision`, expanded strategies and plug-in ecosystem[^4]. |
| Swift Testing | 2024 | `@Suite(.snapshots(record:))` trait, `withSnapshotTesting`, new `record:` modes[^3]. |

Exact per-release dates beyond the repository creation date are not asserted here; see the releases page[^4].

## References

[^1]: Point-Free, "Episode 41: A Tour of Snapshot Testing." https://www.pointfree.co/episodes/ep41-a-tour-of-snapshot-testing
[^2]: Point-Free, witness-oriented / protocol-witness library design series (episodes 33–39). https://www.pointfree.co/episodes/ep39-witness-oriented-library-design
[^3]: Swift Testing integration and recording API, project README. https://github.com/pointfreeco/swift-snapshot-testing#usage
[^4]: Repository, releases, and documentation. https://github.com/pointfreeco/swift-snapshot-testing/releases · https://swiftpackageindex.com/pointfreeco/swift-snapshot-testing/main/documentation/snapshottesting

## Tags

swift, snapshot-testing, screenshot-testing, testing, ios, macos, xctest, swift-testing, ui-testing, pointfree
