# kean/Nuke

> A lean, modular image loading and caching pipeline for Apple platforms, built around a single configurable `ImagePipeline`.

[GitHub repo](https://github.com/kean/Nuke) ·
[Official website](https://kean.blog/nuke) ·
[License: MIT](https://github.com/kean/Nuke/blob/main/LICENSE)

## Overview

Nuke is an image loading system for Apple platforms (iOS, macOS, watchOS, tvOS, visionOS), written and maintained largely by a single author, Alexander Grebenyuk, since 2015[^1]. Its purpose is narrow and well-scoped: fetch an image from a URL (or other source), decode it, optionally process it, cache it in memory and on disk, and hand it to a view — while deduplicating in-flight requests, honoring priorities, and prefetching ahead of scroll. It has stayed in that lane for a decade rather than expanding into a general networking or UI toolkit.

The defining design choice is that almost everything routes through one type, `ImagePipeline`, whose behavior is set by an `ImagePipeline.Configuration`. Loading, coalescing, decoding, processing, memory cache, and disk cache are all pluggable protocols on that configuration. This buys unusual customizability — you can swap the data loader for Alamofire, add WebP/AVIF decoders, or replace the disk cache — at the cost of a larger conceptual surface than "call a function with a URL." For the common case, `LazyImage` (SwiftUI) or `ImageView` (UIKit/AppKit) hide all of it.

The main tension in adopting Nuke is not capability but ecosystem gravity. Kingfisher and SDWebImage are more widely used and have larger contributor bases; Nuke's advantage is a smaller, faster-compiling, more composable core, and its risk is bus-factor and the churn of a codebase that tracks Swift concurrency aggressively.

## Getting Started

Add via Swift Package Manager (the recommended and primary distribution channel; binary frameworks are attached to releases as a fallback)[^2]:

```
https://github.com/kean/Nuke.git
```

Select the modules you need — the core `Nuke`, plus `NukeUI` for views, `NukeExtensions` for `UIImageView` helpers, or `NukeVideo` for short video decoding.

```swift
import NukeUI
import SwiftUI

struct ContentView: View {
    var body: some View {
        LazyImage(url: URL(string: "https://example.com/image.jpeg"))
            .onFailure { print($0) }
    }
}
```

Or drive the pipeline directly with async/await:

```swift
import Nuke

func loadImage(url: URL) async throws -> UIImage {
    let task = ImagePipeline.shared.imageTask(with: url)
    for await progress in task.progress {
        print("\(progress.completed)/\(progress.total)")
    }
    return try await task.image
}
```

## Architecture / How It Works

`ImagePipeline` is the coordinator. An `ImageRequest` describes what to load (a URL or `URLRequest`, a priority, a set of `ImageProcessing` steps, and cache policy). The pipeline resolves each request through a fixed sequence of stages, each backed by a protocol you can replace in `Configuration`:

- **Data loading** — `DataLoading`, by default a wrapper over `URLSession`. This is the seam the Alamofire plugin replaces.
- **Coalescing** — identical in-flight requests are merged so the same URL is fetched once even if fifty cells ask for it simultaneously. Deduplication also covers the decoding and processing stages, keyed by request parameters.
- **Decoding** — `ImageDecoding`, with progressive JPEG support and pluggable decoders for HEIF, WebP, GIF, and AVIF (the latter two via community packages).
- **Processing** — `ImageProcessing` transforms (resize, round corners, blur, custom Core Image filters) run off the main thread; results are cached under the processed key.
- **Caching** — a two-tier system: an in-memory `ImageCache` (LRU, cost-limited, decompressed bitmaps) and an on-disk `DataCache` that stores the original encoded data. The pipeline can also honor the shared `URLCache` (HTTP caching) as a distinct third layer, which is a frequent source of confusion.

Two internal details matter in practice. First, decompression: Nuke forces bitmap decompression off the main thread before handing images to views, which is what prevents frame drops when a decoded-but-not-decompressed image would otherwise be inflated during scrolling. Second, prioritization: `ImagePrefetcher` and per-request priorities let visible cells preempt prefetch work, and priorities can be raised or lowered on a live `ImageTask`.

The codebase tracks Swift's concurrency model closely. Recent versions lean into `async/await` and actor isolation; Nuke 13 requires Swift 6.2 and Xcode 26 and adopts strict concurrency[^3]. This keeps the API modern but means the framework is a moving target across major versions.

## Production Notes

- **Three cache layers, not one.** Memory cache, Nuke's `DataCache`, and the system `URLCache` are independent. If you enable both `DataCache` and `URLCache` you can end up storing image data twice on disk. The common production setup is the aggressive `DataCache` for image data and disabling HTTP disk caching for those requests. Read the caching guide before assuming defaults[^4].
- **Memory cache cost, not count.** `ImageCache` is bounded by total cost (decompressed byte size), not image count, and it responds to memory-pressure notifications. Large full-resolution images can evict many small thumbnails; downsample with a processor before caching if you display thumbnails.
- **Processed vs. original keys.** A resize processor produces a distinct cache entry from the original. Loading the same URL with and without processors, or with differing processor parameters, multiplies cache entries. Keep processor configurations stable across call sites.
- **Concurrency-driven upgrades.** Because Nuke follows Swift concurrency, major-version bumps have carried real migration cost (Sendable annotations, actor isolation, API renames). The repo ships per-version migration guides[^5]; budget time for them rather than treating updates as drop-in.
- **Platform floor moves with Swift.** Nuke 13 raises the minimum to iOS 15 / macOS 12 and Swift 6.2; Nuke 12 targeted iOS 13 and Swift 5.7[^3]. Apps supporting older OSes must pin to an older major.
- **Format plugins are community-maintained.** WebP and AVIF support come from third-party packages of varying activity, not the core. Verify the specific plugin's maintenance status before depending on it for a shipping app.

## When to Use / When Not

**Use when:**
- You want a small, fast-compiling core you can extend by protocol rather than fork.
- You need fine control over the pipeline: custom data loaders, decoders, processors, or cache backends.
- You're on modern Swift/SwiftUI and want first-class `async/await` and `LazyImage`.
- You care about scroll performance and want off-main decompression and coalescing by default.

**Avoid when:**
- You need to support OS versions below Nuke's current floor and don't want to pin to an old major.
- Bus-factor matters for your risk model: development is concentrated on one maintainer.
- You want the largest community and the most Stack Overflow answers — Kingfisher and SDWebImage lead there.
- You need a heavy built-in feature set (extensive built-in transition animations, indicators) out of the box rather than composing it.

## Alternatives

- onevcat/Kingfisher — Swift-native, larger community and more built-in view conveniences; use when ecosystem size and out-of-the-box SwiftUI/UIKit helpers matter more than a minimal core.
- SDWebImage/SDWebImage — the veteran Objective-C-rooted library with the broadest format plugin ecosystem; use when you need mature WebP/GIF/APNG support and maximum third-party integrations.
- ashleymills/... (PINRemoteImage by Pinterest) — battle-tested at scale in a large app; use when you want a library proven under heavy production load, though it is less actively developed.
- Alamofire/AlamofireImage — image loading layered on Alamofire; use when you already standardize on Alamofire and want image caching that shares that networking stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015 | Initial release; UIKit image loading and caching[^1]. |
| 12.0 | 2023 | Swift 5.7 / Xcode 15 baseline; iOS 13 minimum[^3]. |
| 13.0 | 2025 | Swift 6.2 / Xcode 26; strict concurrency; iOS 15 / macOS 12 minimum[^3]. |

The 9–11 era carried the largest API redesigns and the shift toward `async/await`; exact release dates for those majors are not restated here to avoid inaccuracy — consult the migration guides and release notes for specifics[^5].

## References

[^1]: Nuke repository, "Serving Images Since 2015"; repo created 2015-03-11. https://github.com/kean/Nuke
[^2]: Nuke README, Installation — Swift Package Manager recommended, binary frameworks on releases. https://github.com/kean/Nuke#installation
[^3]: Nuke README, Minimum Requirements table (Nuke 13 → Swift 6.2 / Xcode 26 / iOS 15; Nuke 12 → Swift 5.7 / Xcode 15 / iOS 13). https://github.com/kean/Nuke#minimum-requirements
[^4]: Nuke documentation, Caching guide. https://kean-docs.github.io/nuke/documentation/nuke
[^5]: Nuke migration guides. https://github.com/kean/Nuke/tree/main/Documentation/Migrations

## Tags

swift, ios, macos, image-loading, caching, swiftui, apple-platforms, image-processing, progressive-jpeg, webp, prefetching
