# ChartsOrg/Charts

> Core Graphics charting for iOS/tvOS/macOS — the Swift port of MPAndroidChart, distributed as the `DGCharts` framework.

[GitHub repo](https://github.com/ChartsOrg/Charts) ·
[License: Apache-2.0](https://github.com/ChartsOrg/Charts/blob/master/LICENSE)

## Overview

Charts is a UIKit/AppKit charting library for Apple platforms, originally written by Daniel Cohen Gindi as a line-for-line Swift port of Philipp Jahoda's Android library MPAndroidChart[^1]. It began in 2015 as `danielgindi/Charts`; the canonical repository is now `ChartsOrg/Charts` (the older URL still redirects). The stated goal was single-learning-curve parity: the API mirrors the Android original ~95%, so a team shipping the same chart on both platforms writes nearly identical code twice rather than learning two libraries[^1].

The defining tension is timing. For most of the 2015–2022 window this was the de-facto answer to "how do I draw a chart on iOS," because Apple shipped no first-party option and the UIKit drawing story was do-it-yourself Core Graphics. In 2022 Apple introduced **Swift Charts**, a declarative SwiftUI-native framework, which absorbed most new-project mindshare. To avoid a symbol/name clash with Apple's `Charts` module, this library renamed its framework to **DGCharts** in the 5.0 release — the import is now `import DGCharts`, and the CocoaPods spec is `pod 'DGCharts'`[^2]. It remains heavily used (28k+ stars, ~6k forks) but is now a mature, imperative, UIKit-first library competing against a SwiftUI-first Apple framework, and its commit cadence has slowed accordingly (last push 2026-03).

It is Objective-C compatible (the demo project is written in ObjC to prove it), covers iOS 12+, tvOS 12+, and macOS 10.13+, and has a large accumulated issue backlog (~970 open) reflecting both its age and its port-driven design.

## Getting Started

Swift Package Manager (recommended):

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/ChartsOrg/Charts.git", .upToNextMajor(from: "5.1.0"))
]
```

CocoaPods: add `pod 'DGCharts'` (add `pod 'ChartsRealm'` too if using Realm bindings).

A minimal line chart in a UIKit view controller:

```swift
import DGCharts

let chartView = LineChartView(frame: view.bounds)
view.addSubview(chartView)

let entries = (0..<12).map { ChartDataEntry(x: Double($0), y: Double.random(in: 0...100)) }
let set = LineChartDataSet(entries: entries, label: "Revenue")
set.drawCirclesEnabled = false
set.mode = .cubicBezier

chartView.data = LineChartData(dataSet: set)
chartView.animate(xAxisDuration: 0.8)   // build-up animation
```

## Architecture / How It Works

Charts renders with **Core Graphics** into a `UIView` (`NSView` on macOS), not SwiftUI or Metal. Each chart type is a `UIView` subclass (`LineChartView`, `BarChartView`, `PieChartView`, `CombinedChartView`, `ScatterChartView`, `CandleStickChartView`, `BubbleChartView`, `RadarChartView`) that draws in `draw(_:)` on the main thread. The class hierarchy is deep and mirrors the Android original almost exactly.

The data model is three layers: `ChartDataEntry` (a single point) → `ChartDataSet` (a styled series, e.g. `LineChartDataSet`) → `ChartData` (a collection of sets bound to the view via `.data`). Rendering is split into dedicated renderer objects (axis renderer, data renderer, legend renderer, highlighter) that the view composes — a direct translation of MPAndroidChart's renderer architecture. Interaction (pinch-zoom, pan, double-tap-to-zoom, value highlighting) is handled by gesture recognizers wired into the view's transformer, which maps data-space coordinates to pixel-space.

The port relationship is the single most important architectural fact. The library deliberately keeps its shape identical to the Java original so knowledge transfers, but that means it inherits Java-idiomatic patterns (mutable setters everywhere, class-based configuration, no value-type ergonomics) rather than Swift-native ones. There is no meaningful iOS-specific documentation: the README explicitly directs you to read the **MPAndroidChart wiki** and translate mentally[^1]. SwiftUI usage requires your own `UIViewRepresentable` wrapper — there is no first-party SwiftUI API.

## Production Notes

**No native SwiftUI support.** You must hand-roll a `UIViewRepresentable`/`NSViewRepresentable` wrapper and manage data updates through `updateUIView`. This is a common source of stale-render and retain-cycle bugs. If your app is SwiftUI-first and greenfield, Apple's Swift Charts is usually the lower-friction choice.

**Documentation is by proxy.** The canonical reference is the Android wiki; the ObjC/Swift demo projects (`ChartsDemo-iOS`, `ChartsDemo-macOS`) are the real how-to source. Budget time for reading demo code — API discovery via autocomplete alone is painful given the deep hierarchy.

**The rename is a hard upgrade edge.** Moving to 5.x means changing every `import Charts` to `import DGCharts` and the pod name from `Charts` to `DGCharts`[^2]. Mixed dependency graphs where a transitive dependency still references the old module name cause build failures. Read the 5.0 migration notes before bumping.

**Main-thread rendering cost.** All drawing is Core Graphics on the main thread. Very large datasets (tens of thousands of entries), frequent live updates, or many simultaneous charts can drop frames. Mitigations: decimate/downsample before handing data to the chart, disable circle drawing and value labels on dense line sets, and avoid animating on every data refresh.

**Compiler sensitivity.** The README warns that the `master` branch tracks the latest Swift compiler; if you are not on the newest Xcode you should pin a tagged release rather than build from `master`[^1]. Historically, new Swift versions have required coordinated releases.

**Realm coupling is optional and separate.** Realm bindings live in the sibling `ChartsRealm` package, not the core, and must be kept version-compatible with your Realm framework yourself.

**Maintenance tempo.** With ~970 open issues and a slowed commit cadence, expect community-driven answers (Stack Overflow `ios-charts` tag) rather than fast maintainer turnaround on bugs.

## When to Use / When Not

**Use when:**
- You have a UIKit/AppKit codebase and need rich, interactive charts (zoom, pan, highlight, combined chart types) today.
- You already ship MPAndroidChart on Android and want matching visuals and near-identical code across platforms.
- You need chart types or interaction depth (candlestick, radar, bubble, dual axes, combined charts) that Swift Charts does not cover as fully.
- You must support older OS versions (iOS 12+) that predate Swift Charts (iOS 16+).

**Avoid when:**
- You are building a new SwiftUI app targeting iOS 16+ — Apple's Swift Charts is native, declarative, and first-party.
- You want strong iOS-specific docs and an actively fast-moving maintainer — this is a mature, slower project documented by proxy.
- Your dataset is very large or updates at high frequency and you cannot pre-decimate — main-thread Core Graphics will bottleneck.
- You want a value-type, Swift-idiomatic API rather than a Java-shaped class hierarchy.

## Alternatives

- apple/swift-charts (Swift Charts) — use instead when you are SwiftUI-first on iOS 16+ and want a native, declarative, Apple-maintained framework.
- PhilJay/MPAndroidChart — the Android original; use on Android, or as the shared design reference when shipping both platforms.
- danielgindi/Charts — the historical name for this same repository (now redirects to ChartsOrg/Charts); not a separate library.
- danielgindi/ios-charts is not it — the correct pod is `DGCharts`; older `ios-charts`/`Charts` pod names are stale or belong to unrelated projects[^2].
- SwiftUICharts / community SwiftUI chart packages — use for simple SwiftUI charts when you want a lightweight dependency and don't need this library's interaction depth.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-03 | `danielgindi/Charts` created as a Swift port of MPAndroidChart[^1]. |
| 3.0 | 2017 | Swift 3 rewrite; broad API changes tracking the language. |
| 4.0.0 | 2021 | Sync milestone with MPAndroidChart; module still named `Charts`. |
| 5.0.0 | 2023 | Framework renamed to **DGCharts** to avoid clash with Apple's Swift Charts; breaking migration[^2]. |
| 5.1.0 | 2024 | Carthage prebuilt binaries, SPM `upToNextMajor(from: "5.1.0")`[^1]. |

## References

[^1]: ChartsOrg/Charts README — project origin, MPAndroidChart parity, compiler/branch guidance, install instructions. https://github.com/ChartsOrg/Charts
[^2]: Charts 5.0.0 release / migration notes — DGCharts rename and breaking changes. https://github.com/ChartsOrg/Charts/releases/tag/5.0.0

## Tags

swift, ios, macos, tvos, charts, data-visualization, uikit, core-graphics, objective-c, mpandroidchart-port, cocoapods, apache-2.0
