# airbnb/lottie-ios

> Renders After Effects vector animations natively on Apple platforms from designer-exported JSON.

[GitHub repo](https://github.com/airbnb/lottie-ios) ·
[Official website](https://lottie.airbnb.tech) ·
[License: Apache-2.0](https://github.com/airbnb/lottie-ios/blob/master/LICENSE)

## Overview

Lottie is a library that plays vector animations authored in Adobe After Effects, exported to a JSON format (Bodymovin) by a designer, and rendered natively at runtime with no image sequences or GIFs[^1]. The iOS variant covers iOS, macOS, tvOS, and visionOS; sibling projects `airbnb/lottie-android` and `airbnb/lottie-web` render the same JSON on their platforms, so a single export ships to all three. This "design once, render everywhere from the source vector" workflow is the whole point: animations stay small (JSON, often a few KB), scale without raster artifacts, and can be recolored, scrubbed, looped, or reversed at runtime.

The defining tension is **fidelity vs. rendering performance**, and it maps directly onto the library's two rendering engines. After Effects is a deep tool; Lottie implements a subset of it. Some effects, expressions, and layer features that a motion designer uses freely have no Lottie equivalent, so what plays perfectly in After Effects can render wrong — or not at all — on device. The gap between "what the designer made" and "what Lottie supports" is the single most common source of friction in production, and it is a communication problem as much as a technical one[^2].

Lottie began at Airbnb in Objective-C and was rewritten in Swift in the 3.x line[^3]. It is broadly stable, actively maintained (frequent pushes, 26k+ stars, Apache-2.0), and effectively the default for JSON-driven motion on Apple platforms.

## Getting Started

Swift Package Manager (recommended via the `lottie-spm` mirror, not the main repo — see Production Notes):

```swift
// Package.swift
.package(url: "https://github.com/airbnb/lottie-spm.git", from: "4.6.1")
```

CocoaPods: `pod 'lottie-ios'`. Carthage: `github "airbnb/lottie-ios" "master"`.

UIKit:

```swift
import Lottie

let animationView = LottieAnimationView(name: "loader") // loader.json in bundle
animationView.contentMode = .scaleAspectFit
animationView.loopMode = .loop
view.addSubview(animationView)
animationView.play()
```

SwiftUI:

```swift
import Lottie

struct LoadingView: View {
    var body: some View {
        LottieView(animation: .named("loader"))
            .looping()
    }
}
```

## Architecture / How It Works

A `.json` export is parsed into a model tree of layers (shape, image, text, precomp, null), each with keyframed transform and property values on a shared timeline. Rendering that model is where the two engines diverge[^4]:

1. **Main Thread engine** — the original renderer. Each frame is computed in Swift and drawn with Core Graphics / `CAShapeLayer`s on the main thread. It supports the widest set of After Effects features (masks, mattes, most effects, dynamic runtime property overrides). The cost is CPU: complex animations recompute every frame on the main thread and can drop frames or block scrolling.

2. **Core Animation engine** (added in the 4.x line) — converts the model into `CALayer`/`CAAnimation` objects once, then hands them to the system render server, which animates them off the main thread. Much cheaper per frame, but it only supports the subset of features expressible as native Core Animation, so certain effects and dynamic property changes are unavailable or ignored.

The `RenderingEngineOption` controls the choice. `.automatic` (a common default) inspects the animation via a compatibility check and uses Core Animation when the animation is fully supported, silently falling back to the Main Thread engine when it is not. This is convenient but means the *same code* can run on either engine depending on the specific JSON — a subtlety that matters when a rendering bug appears only for some animations.

Runtime customization goes through **value providers** and **keypaths**: you address a node inside the animation by its layer/property path and supply a provider (color, float, point) that overrides the exported value — how "recolor at runtime" and progress-driven scrubbing are implemented. **dotLottie** (`.lottie`) is a supported zip container bundling one or more animations plus assets, loaded through an async API[^1].

## Production Notes

**The source repo is large (300+ MB) — do not add it directly via SPM.** SPM always clones full git history, so depending on `lottie-ios.git` pulls the entire repo including animation assets and history. Use `airbnb/lottie-spm` instead: it is under 500 KB and points at a precompiled XCFramework (~8 MB) attached to each release[^5]. This is the most impactful, least-obvious setup decision.

**Feature support is a subset of After Effects, and the failure mode is silent.** Unsupported effects/expressions render as missing or wrong rather than erroring. Establish a designer workflow constrained to the documented supported-features list, and preview every animation on a real device before shipping[^2]. After Effects *expressions* in particular are not evaluated.

**`.automatic` engine selection can produce inconsistent behavior.** An animation that uses a dynamically-set color or an unsupported effect will fall back to the Main Thread engine, quietly changing performance characteristics. If you rely on runtime value providers for a specific animation, verify which engine it actually runs on; pin the engine explicitly (`LottieConfiguration`) when consistency matters.

**Performance scales with complexity, not file size.** A tiny JSON with many nested precomps, masks, or per-frame shape recomputation can be expensive on the Main Thread engine. For loaders and busy scroll views, prefer Core Animation-compatible animations. Profile with Instruments; do not assume "small JSON = cheap".

**XCFramework code signing.** Release XCFrameworks are self-signed since 4.4.0; verify the published fingerprint if your pipeline checks binary provenance[^6]. Lottie collects no data and ships a privacy manifest (`PrivacyInfo.xcprivacy`) for App Privacy details.

**Snapshot tests are the contract.** Rendering changes are gated by snapshot tests that must run on a specific simulator (iPhone 8), which matters if you contribute or vendor patches.

## When to Use / When Not

**Use when:**
- A motion designer owns the animation and can export Bodymovin JSON; you want to ship it without hand-coding.
- You need the same animation on iOS, Android, and web from one source.
- You need runtime control: recolor, scrub to progress, loop segments, react to state.
- Assets must stay resolution-independent and small (onboarding, loaders, celebratory/empty states).

**Avoid when:**
- The animation leans on After Effects effects/expressions Lottie doesn't support — the export will look wrong.
- You need interactive, stateful animation logic (branching state machines) — Lottie plays timelines, it is not a state-machine runtime; consider Rive.
- The motion is trivial (a spinner, a fade) — native `UIView`/SwiftUI animation or a `CAAnimation` avoids the dependency entirely.
- You cannot afford a designer-in-the-loop workflow and QA on device.

## Alternatives

- rive-app/rive-ios — interactive animations with state machines and a dedicated editor; use Rive when you need branching/interactive motion rather than fixed timelines.
- airbnb/lottie-android — the Android sibling; use when you need the same Bodymovin JSON on Android.
- airbnb/lottie-web — the web/JS renderer; use for the same animations in a browser.
- SDWebImage/SDWebImageLottieCoder — decode Lottie inside an image-loading pipeline; use when animations arrive as remote image URLs.
- Apple Core Animation / SwiftUI animations — use when the motion is simple enough to express in code and a dependency isn't justified.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x–2.x | 2017 | Initial public releases, written in Objective-C[^3]. |
| 3.0 | 2020 | Full rewrite in Swift; new model/rendering API[^3]. |
| 4.0 | 2022 | Core Animation rendering engine added alongside Main Thread engine; `.automatic` selection[^4]. |
| 4.4.0 | 2024 | Code-signed XCFramework distribution[^6]. |
| 4.6.1 | 2026 | Current release line; SPM via `lottie-spm`[^5]. |

## References

[^1]: Lottie iOS README and documentation. https://airbnb.io/lottie/
[^2]: Lottie supported After Effects features / feature-compatibility documentation. https://airbnb.io/lottie/#/supported-features
[^3]: Lottie iOS releases and repository history. https://github.com/airbnb/lottie-ios/releases
[^4]: Lottie iOS rendering engines (Main Thread vs. Core Animation). https://airbnb.io/lottie/#/ios?id=rendering-engines
[^5]: `airbnb/lottie-spm` — lightweight SPM distribution pointing at the release XCFramework. https://github.com/airbnb/lottie-spm
[^6]: Lottie iOS README, "Security" — signed XCFramework since 4.4.0. https://github.com/airbnb/lottie-ios#security

## Tags

swift, ios, macos, animation, after-effects, bodymovin, vector-graphics, dotlottie, mobile, apple-platforms, airbnb
