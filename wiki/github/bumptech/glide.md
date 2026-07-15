# bumptech/glide

> An Android image loading and caching library built around one goal: keep list scrolling smooth by never blocking the main thread and never thrashing the garbage collector.

[GitHub repo](https://github.com/bumptech/glide) ·
[Official website](https://bumptech.github.io/glide/) ·
[License: BSD-2-Clause (part MIT, part Apache-2.0)](https://github.com/bumptech/glide/blob/master/LICENSE)

## Overview

Glide is a media loading library for Android that wraps network fetch, bitmap
decoding, in-memory and disk caching, resource pooling, and lifecycle-aware
request management behind a fluent `Glide.with(...).load(...).into(...)` API[^1].
It was written by Sam Judd (originally at Bump Technologies, later Google) and
has been public since 2013. It is not an official Google product, though it is
widely used inside Google apps.

The defining design goal is stated in the tagline: smooth scrolling. Where a
naive `ImageView` populate stutters because decoding full-resolution bitmaps on
the main thread stalls the frame, and because allocating a fresh `Bitmap` per
item triggers GC pauses, Glide attacks both. It downsamples images to the target
view size at decode time, recycles bitmaps through a pool so steady-state
scrolling allocates almost nothing, and ties every request to an Activity or
Fragment lifecycle so off-screen work is paused, resumed, or cancelled
automatically[^2]. Native support for animated GIFs and video stills — not just
static images — is a historical differentiator from Square's Picasso.

The tradeoff is size and surface area. Glide carries more method count and a
larger runtime than the minimalist alternatives, its configuration API is broad
enough to be its own learning curve, and the annotation-processor-generated
extension API adds a build-time step. In the Compose / Kotlin-coroutines era it
also competes with Coil, which many new projects reach for first. Glide's
answer is a very large installed base, aggressive memory behavior tuned over a
decade, and format coverage that the lighter libraries do not match.

## Getting Started

Gradle (`com.github.bumptech.glide:glide`), latest is the 5.0.x line[^3]:

```gradle
repositories {
  google()
  mavenCentral()
}
dependencies {
  implementation 'com.github.bumptech.glide:glide:5.0.9'
  annotationProcessor 'com.github.bumptech.glide:compiler:5.0.9' // or ksp(...)
}
```

The core call, safe to make from an Activity, Fragment, or view:

```java
Glide.with(fragment)
    .load("https://example.com/image.jpg")
    .placeholder(R.drawable.loading_spinner)
    .centerCrop()
    .into(imageView);
```

R8/ProGuard rules are bundled in the AAR and consumed automatically — no manual
keep rules for the core library.

## Architecture / How It Works

A load request flows through three cache tiers before it ever hits the network:

1. **Active resources** — a weak-reference map of bitmaps currently displayed on
   screen. Prevents a visible image from being evicted and re-decoded.
2. **Memory cache** — an LRU cache (`LruResourceCache`) of decoded, ready-to-draw
   resources.
3. **Disk cache** — by default the *transformed, resized* result is written to a
   `DiskLruCache` (the implementation is derived from Jake Wharton's
   `DiskLruCache`[^1]), keyed by a cache signature. The strategy is configurable
   (`DiskCacheStrategy.DATA` / `RESOURCE` / `ALL` / `AUTOMATIC`) to trade disk
   space against re-transform cost.

Misses go to the fetch + decode pipeline. Fetching is pluggable: the default is a
custom `HttpUrlConnection` `ModelLoader`, with optional `okhttp3-integration` and
`volley-integration` artifacts to swap in those stacks. Decoding downsamples via
`BitmapFactory.Options.inSampleSize` so a 4000px source destined for a 400px view
never fully materializes in memory. Decoded bitmaps are drawn from and returned to
a **`BitmapPool`**, which is the mechanism that keeps steady-state scrolling
allocation-free.

**Lifecycle integration** is implemented by attaching an invisible
`SupportRequestManagerFragment` (or lifecycle observer) to the host, so
`Glide.with(activity)` requests pause on `onStop` and cancel on destroy. This is
why *which* context you pass matters (see Production Notes).

**Generated API.** Glide's annotation processor turns an `@GlideModule`
`AppGlideModule` subclass into a generated `GlideApp` entry point and lets library
authors contribute options via `@GlideExtension`. This is where global defaults
(default disk-cache strategy, default bitmap format, custom `ModelLoader`s) are
registered. It requires wiring the `compiler` annotation processor (or KSP).

**Compose.** A separate `com.github.bumptech.glide:compose` artifact provides a
`GlideImage` composable for Jetpack Compose, distinct from the classic
`ImageView`-based API.

## Production Notes

**Context choice is a correctness issue, not a style choice.** Passing an
Activity/Fragment to `Glide.with` scopes the request to that lifecycle; passing
`applicationContext` opts out of lifecycle management entirely and can leak or
load into a dead view. Loading into a view attached to a destroyed Activity is a
recurring crash source.

**Bitmap format and memory.** Glide historically defaulted to `RGB_565` (half the
bytes of `ARGB_8888`) to favor memory over quality, which produced visible
banding on gradients versus Picasso's `ARGB_8888` default; the modern default is
`ARGB_8888`. If you see banding or unexpected memory profiles, check the
configured `DecodeFormat` explicitly rather than assuming the default[^2].

**Disk cache stores the transformed result by default.** Loading the same URL at
two different sizes/transforms produces two disk entries. Loading the same image
that you also need at original resolution elsewhere can mean it is fetched twice
unless you set `DiskCacheStrategy.ALL` or `.DATA`. Cache keys incorporate
transformations and any `.signature()` you supply — mutable remote images that
reuse a URL need a changing signature or they will serve stale bytes.

**`into()` vs `submit()`/`preload()`.** `into(ImageView)` is main-thread only.
For background/synchronous fetches use `submit().get()` (never on the main
thread) or `downloadOnly`. Preloading with `RecyclerViewPreloader` is the
standard trick for making fast fling scrolling feel instant.

**Version 5 migration.** The 5.0.0 stable release (2025) followed an unusually
long RC period — `v5.0.0-rc01` was tagged in 2023, nearly two years before the
final[^3]. It raised minimum toolchain requirements and adjusted defaults; treat
it as a real migration, not a patch bump, and read the release notes for the
`RequestOptions` and module-registration changes before upgrading a large app.

**Transformations.** Rounded/circle images plus `TransitionDrawable` cross-fades
and animated GIFs are a known conflict; use a `BitmapTransformation`
(`.circleCrop()`) or `.dontAnimate()` rather than fighting `CircleImageView`
subclasses. The community `wasabeef/glide-transformations` library covers blur,
grayscale, and similar effects.

## When to Use / When Not

**Use when:**
- You have long scrolling lists (RecyclerView/ListView) where jank and GC pauses
  are the problem you are actually solving.
- You need animated GIF or video-frame loading alongside static images.
- You want fine-grained cache-strategy, downsampling, and transformation control.
- You maintain an existing `ImageView`-based (non-Compose) codebase.

**Avoid when:**
- You are starting a Kotlin-first, Compose-only app — Coil integrates more
  naturally and is lighter.
- You want the smallest possible dependency and load only a handful of static
  images — Picasso or Coil are simpler.
- You are extremely memory-constrained with huge image counts — Fresco's
  off-heap pipeline may fit better.

## Alternatives

- coil-kt/coil — Kotlin/coroutines-native, first-class Compose support, smaller; use instead when starting a modern Kotlin or Compose app.
- square/picasso — minimalist fluent API from the same lineage; use instead when you only load static images and want the smallest footprint.
- facebook/fresco — image pipeline with off-heap bitmap storage and its own Drawee views; use instead when memory pressure from many large images dominates.
- google/accompanist — historically bridged Glide/Coil into early Compose; superseded by native Coil Compose, relevant only for legacy code.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-07-08 | Repository created; `HttpUrlConnection` stack, bitmap pooling. |
| 4.0.0 | 2017-08-01 | Generated API (`@GlideModule`/`GlideApp`), `RequestOptions`, `RequestBuilder` overhaul[^3]. |
| 4.16.0 | 2023-08-21 | Final 4.x feature release; Compose integration matured in parallel (`compose-alpha`). |
| 5.0.0-rc01 | 2023-09-26 | First 5.0 release candidate; ~2-year RC period follows. |
| 5.0.0 | 2025-08-31 | 5.x stable line; raised minimums, updated defaults[^3]. |
| 5.0.9 | 2026-07-11 | Current 5.0.x patch at time of writing. |

## References

[^1]: bumptech/glide README — API, cache design, and DiskLruCache/GIF-decoder attributions. https://github.com/bumptech/glide/blob/master/README.md
[^2]: Glide documentation — caching, downsampling, lifecycle, and configuration. https://bumptech.github.io/glide/
[^3]: Glide releases and tags (dates verified via GitHub API). https://github.com/bumptech/glide/releases

## Tags

android, java, image-loading, caching, bitmap, recyclerview, gif, media, mobile, ui, performance
