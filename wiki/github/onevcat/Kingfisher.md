# onevcat/Kingfisher

> Pure-Swift image downloading and caching, glued to UIKit/AppKit/SwiftUI view extensions with a `kf` namespace.

[GitHub repo](https://github.com/onevcat/Kingfisher) ·
[License: MIT](https://github.com/onevcat/Kingfisher/blob/master/LICENSE)

## Overview

Kingfisher is a single-purpose library for Apple platforms: fetch a remote
image from a URL, decode it, cache it in memory and on disk, and set it on a
view. First published in 2015 by Wei Wang (onevcat)[^1], it predates
SwiftUI and Swift Concurrency and has been kept current across every major
Swift transition since. Its API surface is deliberately narrow — the
maintainer states repeatedly that the goal is to stay "lightweight" and not
grow into a general media framework.

The defining design choice is the `kf` extension namespace. Instead of calling
free functions, you write `imageView.kf.setImage(with: url)`; the `.kf`
property is a `KingfisherWrapper` that hangs the whole API off existing UIKit,
AppKit, and WatchKit view types without polluting their method space. The
same idea is repackaged three ways: the `kf` extension (imperative), the `KF`
builder (chained), and `KFImage` (SwiftUI view). All three drive the same
underlying `KingfisherManager`, so behavior is consistent regardless of entry
point.

The central tension is scope discipline versus feature pressure. Kingfisher
resists becoming SDWebImage (broad format plugins, Objective-C core) but has
still accreted processors, transitions, indicators, prefetching, Low Data Mode,
and — as of v8 — Live Photo loading[^2]. Each addition is small, but the option
set (`KingfisherOptionsInfo`) is now large enough that two callers can produce
subtly different cache behavior without noticing.

## Getting Started

Swift Package Manager (add `https://github.com/onevcat/Kingfisher.git`, "Up to
Next Major" from `8.0.0`), or CocoaPods `pod 'Kingfisher', '~> 8.0'`.

```swift
import Kingfisher

// UIKit — download, cache to memory + disk, display
let url = URL(string: "https://example.com/image.png")
imageView.kf.setImage(with: url)
```

```swift
// A processor pipeline: downsample to the view size, then round the corners.
// Cache key = URL + processor identifier, so this variant caches separately.
let processor = DownsamplingImageProcessor(size: imageView.bounds.size)
             |> RoundCornerImageProcessor(cornerRadius: 20)
imageView.kf.setImage(
    with: url,
    placeholder: UIImage(named: "placeholder"),
    options: [.processor(processor), .scaleFactor(UIScreen.main.scale),
              .transition(.fade(1)), .cacheOriginalImage])
```

```swift
// SwiftUI — same engine behind KFImage
var body: some View { KFImage(url) }
```

## Architecture / How It Works

`KingfisherManager` is the coordinator. A request flows: check `ImageCache`
(memory first, then disk), and on a miss hand off to `ImageDownloader`, then
run the result through the processor chain, then write back to both cache
tiers. The view extension is a thin adapter that resolves the target view,
issues the manager call, and applies the placeholder/indicator/transition.

`ImageCache` is two independent stores. `MemoryStorage` is an `NSCache`-style
in-memory store that is purged on memory-pressure notifications;
`DiskStorage` is a file-backed store with per-entry expiration and a total
size limit, swept periodically. Both are keyed by a computed cache key, which
is the source's `cacheKey` (URL by default) combined with the processor's
`identifier`. This is the single most important internal detail to understand:
**a different processor means a different cache entry**, so the same image
downsampled two ways is stored twice.

`ImageProcessor` is a protocol with an `identifier` and a transform; the `|>`
operator composes processors left to right. `Source` abstracts where bytes
come from (`.network(Resource)` or `.provider(ImageDataProvider)`), which is
how local-data and custom loaders reuse the same pipeline.

The download side deduplicates: concurrent requests for the same URL share one
`SessionDataTask`, and each caller gets its own cancel token. Cancelling one
caller (e.g. a reused table cell) does not cancel the shared download for
others. SwiftUI's `KFImage` wraps this in an `ObservableObject` binder that
ties the request lifecycle to view identity.

## Production Notes

- **Cache key + processor bloat.** Because the disk key folds in the processor
  identifier, apps that apply view-size-dependent `DownsamplingImageProcessor`
  can generate many disk entries for one source image (one per distinct size).
  Set an explicit `.cacheOriginalImage` and/or a stable processor, and cap
  `diskStorage.config.sizeLimit`, or disk usage grows unbounded.
- **Memory vs. downsampling.** Setting a full-resolution image into a small
  view keeps the decoded full-size bitmap in the memory cache. In list/grid UI
  this is the usual OOM cause; `DownsamplingImageProcessor` sized to the view
  is the standard fix and should be treated as default, not optional.
- **Cell reuse.** `setImage` on a reused cell cancels the prior in-flight set
  for that view, but you still want to clear or placeholder in
  `prepareForReuse` to avoid a flash of the recycled image.
- **SwiftUI `KFImage` identity.** `KFImage` reloads when its view identity
  changes; in `List`/`LazyVStack` an unstable `.id()` or changing options
  object can trigger redundant reloads. Cancel-on-disappear behavior is
  configurable and has changed across versions — pin it explicitly.
- **Swift 6 strict concurrency.** v8 is prepared for Swift 6 strict-concurrency
  mode[^2]; migrating an app to that mode still surfaces sendability warnings
  around custom `ImageProcessor`/`CacheSerializer` types you pass in.
- **Disk reads are async by default.** Synchronous disk hits require
  `.loadDiskFileSynchronously`; without it, an image already on disk may
  render one frame late (visible flicker) rather than immediately.

## When to Use / When Not

**Use when:**
- You ship on Apple platforms and want URL-to-view image loading with sane
  memory+disk caching out of the box.
- You want one library covering UIKit, AppKit, and SwiftUI with a consistent
  API and processor pipeline.
- You need cache-key control, expiration, prefetching, or Low Data Mode
  without hand-rolling `URLSession` + a cache.

**Avoid when:**
- You need heavy format support or a plugin ecosystem (animated WebP, custom
  coders) — SDWebImage's coder plugins are broader.
- Your needs are trivial and you target recent OS versions — SwiftUI's
  built-in `AsyncImage` avoids a dependency entirely (at the cost of no real
  caching control).
- You want the leanest possible Swift-first core with minimal API — Nuke is
  smaller in scope.

## Alternatives

- SDWebImage/SDWebImage — Objective-C core with a wide coder/plugin ecosystem;
  use when you need animated WebP, custom decoders, or long-tail format support.
- kean/Nuke — leaner, Swift-first, performance-focused; use when you want a
  minimal, composable pipeline over Kingfisher's broader convenience surface.
- Apple's AsyncImage (SwiftUI, built in) — use when caching control does not
  matter and you want zero dependencies on modern OS versions.
- kean/NukeUI — Nuke's view layer (`LazyImage`/`AsyncImage` equivalent); use
  when you have standardized on the Nuke stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015 | Initial release; `kf` extension for UIKit image views[^1]. |
| 3.0 | 2016 | Swift 3 rewrite. |
| 5.0 | 2019 | Major API redesign: `Source`, processor `|>` pipeline, `Result`-based callbacks[^3]. |
| 6.0 | 2020 | `KFImage` for SwiftUI; `KF` builder. |
| 7.0 | 2021 | Memory/disk cache reworks; async/await support. |
| 8.0 | 2024 | Swift 6 / strict concurrency prep, visionOS, Live Photo loading[^2]. |

## References

[^1]: onevcat/Kingfisher repository, created 2015-04-06. https://github.com/onevcat/Kingfisher
[^2]: Kingfisher 8.0 features and requirements (Swift 6 concurrency, visionOS, Live Photo), README and migration guide. https://swiftpackageindex.com/onevcat/kingfisher/master/documentation/kingfisher/migration-to-8
[^3]: Kingfisher documentation home (processor pipeline, cache, downloader). https://swiftpackageindex.com/onevcat/kingfisher/master/documentation/kingfisher

## Tags

swift, ios, macos, image-loading, image-cache, image-processing, swiftui, uikit, networking, apple-platforms
