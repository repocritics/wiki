# coil-kt/coil

> Coroutine-based image loading for Android and Compose Multiplatform, built on Kotlin, Coroutines, and Okio.

[GitHub repo](https://github.com/coil-kt/coil) ·
[Official website](https://coil-kt.github.io/coil/) ·
[License: Apache-2.0](https://github.com/coil-kt/coil/blob/main/LICENSE.txt)

## Overview

Coil ("**Co**routine **I**mage **L**oader") is an image loading library first released for Android in 2020[^1]. Its pitch has always been narrow and honest: fetch, decode, cache, and display images with a Kotlin-first API that assumes Coroutines are already in the app. It depends only on Kotlin, Coroutines, and Okio, which keeps its method count and transitive footprint smaller than the older Java-era loaders it displaced.

With version 3 (2025) the project stopped being Android-only and became a Compose Multiplatform library, running on Android, JVM, iOS, macOS, JS, and WASM[^2]. This is the defining tension of the current codebase: the same library is now both the pragmatic default for a plain Android `ImageView` app *and* a multiplatform Compose dependency, and the two audiences pull the API in different directions. The v3 rewrite also changed the Maven coordinates (`io.coil-kt.coil3`) and made the network stack pluggable, which means most upgrade guides you find online are for a version whose imports no longer exist.

As of 2026 it is actively maintained (roughly 11.9k stars, frequent commits) and is, for Compose-first Android projects, the most commonly recommended loader — largely because Google's own Jetpack Compose samples and much of the ecosystem lean on it rather than the older Glide/Picasso lineage.

## Getting Started

```kotlin
// build.gradle.kts — Compose + a network backend (OkHttp or Ktor)
implementation("io.coil-kt.coil3:coil-compose:3.5.0")
implementation("io.coil-kt.coil3:coil-network-okhttp:3.5.0")
```

```kotlin
// Compose: the common case
import coil3.compose.AsyncImage

AsyncImage(
    model = "https://example.com/image.jpg",
    contentDescription = null,
)
```

```kotlin
// Imperative / non-Compose: build a request against an ImageLoader
import coil3.ImageLoader
import coil3.request.ImageRequest

val loader = ImageLoader(context)
val request = ImageRequest.Builder(context)
    .data("https://example.com/image.jpg")
    .target { drawable -> imageView.setImageDrawable(drawable.asDrawable(resources)) }
    .build()
loader.enqueue(request)
```

## Architecture / How It Works

The core is an `ImageLoader`, a singleton-by-convention object that owns the memory cache, disk cache, and a shared request pipeline. Individual loads are `ImageRequest`s enqueued against it. The Compose `AsyncImage` / `rememberAsyncImagePainter` layer is a thin adapter that binds a request's lifecycle to composition and recomposition.

An `ImageRequest` flows through an interceptor chain (the same architectural shape as an OkHttp call): mapper/keyer resolution → memory cache lookup → fetcher (produces a source from the `data`, e.g. a URL, `File`, `Uri`, resource id, or `ByteArray`) → decoder (turns the source into an image) → transformations → memory/disk cache write. Every stage is a registered `ComponentRegistry` entry, so support for a new source type or format is added by registering a `Fetcher.Factory` or `Decoder.Factory` rather than subclassing.

Two v3 decisions matter in practice:

- **Pluggable networking.** Coil 3 does not hard-depend on OkHttp. You add `coil-network-okhttp` or `coil-network-ktor` explicitly. On non-JVM targets (iOS/WASM) Ktor is the path. If you forget the network module, remote URLs silently fail to load with no compile error — the fetcher for `http(s)` simply isn't registered.
- **`coil3.PlatformContext`.** The Android `Context` is abstracted so common Compose code can construct requests. On Android it still *is* a `Context`; on other platforms it is a platform shim.

Downsampling, `Bitmap` pooling behavior, and hardware bitmaps are Android-specific and handled in the Android artifact; the multiplatform core stays free of `android.graphics`. Coil is R8/ProGuard-friendly and ships its own consumer rules.

## Production Notes

- **The coordinate/import break is the biggest upgrade cost.** Migrating 1.x/2.x → 3.x is not a version bump: the group id changed to `io.coil-kt.coil3`, packages moved to `coil3.*`, and OkHttp moved out of the core. Expect a mechanical but wide find-and-replace, plus adding the network module you previously got for free[^2].
- **Missing network module = silent no-op.** Because fetchers are registered, not compiled in, omitting `coil-network-okhttp`/`coil-network-ktor` yields blank images at runtime rather than a build failure. This is the single most common "it works locally but not in the new module" report.
- **Disk cache is single-instance.** The `DiskCache` (Okio-based, journaled) assumes one instance per directory per process. Instantiating multiple `ImageLoader`s pointing at the same cache directory, or sharing it across processes, corrupts the journal. Use a single app-wide loader.
- **Memory cache sizing.** The default memory cache is a share of available app memory. On image-heavy grids you often want to raise it, but oversizing it competes with the rest of the app and can worsen GC pressure — measure with real image sizes, not thumbnails.
- **`AsyncImage` and layout.** With unbounded constraints (e.g. a `Column` with no fixed height) `AsyncImage` needs a `Modifier.size`/`aspectRatio` or it can measure to zero before the image resolves. This is a recurring "image doesn't show" cause that is a layout issue, not a Coil bug.
- **Caching semantics.** Cache keys derive from the `data` plus size and transformations. Two requests for the same URL at different target sizes are different disk/memory entries; a signed URL whose query string changes each load defeats caching entirely.
- **GIF/SVG/video need extra artifacts.** Animated GIF (`coil-gif`), SVG (`coil-svg`), and video frames (`coil-video`) are separate modules with their own decoders you must register.

## When to Use / When Not

**Use when:**
- Your app is Kotlin + Coroutines and (increasingly) Compose — Coil's API assumes both.
- You want a small dependency footprint and clean R8 behavior.
- You need one image loader across Compose Multiplatform targets.
- You want a modern, actively maintained loader rather than the maintenance-mode Java-era options.

**Avoid when:**
- You are on a Java-only codebase with no Coroutines — the ergonomics fight you; Glide or Picasso fit better.
- You need Fresco's aggressive off-heap (`ashmem`) bitmap management for very large image sets on low-memory Android devices.
- You are pinned to Coil 1.x/2.x and cannot absorb the v3 coordinate/import migration right now.

## Alternatives

- bumptech/glide — mature Java Android loader, widest format/transform ecosystem; use when you need battle-tested Android-only behavior and RequestBuilder flexibility.
- square/picasso — minimal, simple, effectively feature-frozen; use for small legacy Android apps where "just load a URL" is the whole requirement.
- facebook/fresco — off-heap bitmap pool and progressive JPEG; use when memory pressure from large images on Android is the dominant constraint.
- Kamel-Media/Kamel — Compose Multiplatform image loader; use as a Coil 3 alternative if you prefer a smaller, KMP-native-from-day-one design.
- qdsfdhvh/compose-imageloader — another KMP Compose loader; use when you want a lightweight multiplatform option outside the Coil lineage.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2020-11 | First stable release. Android-only, OkHttp-based, `ImageView` targets[^1]. |
| 2.0 | 2022-05 | API cleanup, improved caching, first-class Jetpack Compose support[^3]. |
| 3.0 | 2025 | Compose Multiplatform rewrite. New `io.coil-kt.coil3` coordinates, `coil3.*` packages, pluggable OkHttp/Ktor networking[^2]. |
| 3.5.x | 2026 | Current line; multiplatform targets incl. iOS/JS/WASM, ongoing decoder/transform work. |

## References

[^1]: Coil releases — version history and tags. https://github.com/coil-kt/coil/releases
[^2]: Coil 3 upgrade guide — coordinate change, package moves, and pluggable networking. https://coil-kt.github.io/coil/upgrading_to_coil3/
[^3]: Coil documentation — getting started and API reference. https://coil-kt.github.io/coil/getting_started/

## Tags

kotlin, android, compose-multiplatform, image-loading, coroutines, okio, jetpack-compose, caching, kotlin-multimedia, mobile
