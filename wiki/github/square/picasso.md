# square/picasso

> An image downloading and caching library for Android — mature, minimal, and now officially deprecated in favor of Coil.

[GitHub repo](https://github.com/square/picasso) ·
[Official website](https://square.github.io/picasso/) ·
[License: Apache-2.0](https://github.com/square/picasso/blob/master/LICENSE.txt)

## Overview

Picasso is an Android image-loading library from Square, first open-sourced in 2013[^1]. Its purpose is narrow and well-defined: fetch an image from a URL (or resource, file, or content URI), decode it, cache it, transform it, and set it on an `ImageView` — while handling the parts Android makes painful, namely off-main-thread decoding, view recycling in lists, and request cancellation. For most of the 2010s it was one of the two or three default answers to "how do I load an image in an Android app," alongside Glide and later Fresco.

The library's defining trait is minimalism. The public surface is essentially one fluent chain — `Picasso.get().load(url).into(view)` — and the internals deliberately delegate networking and HTTP caching to OkHttp rather than reimplementing them. This made Picasso small and easy to reason about, at the cost of features that competitors shipped: it has no animated GIF support, no first-class Jetpack Compose integration, and a disk-cache story that surprises people (see Production Notes).

As of the current README, **Picasso is deprecated**[^2]. Square recommends Coil for new projects and advises migrating existing ones, particularly Compose UI codebases. Existing releases keep working, but no new public releases to Maven Central are planned; the repository is not archived and still sees occasional internal-legacy changes, but it should be treated as end-of-life. The last published release is 2.8. This context is essential: the library is technically sound and still functional, but choosing it for new work in 2026 means adopting something its own authors have sunset.

## Getting Started

Gradle (Groovy DSL):

```groovy
implementation 'com.squareup.picasso:picasso:2.8'
```

Picasso requires at minimum Java 8 and Android API 21[^1]. Basic load into an `ImageView`:

```kotlin
Picasso.get()
    .load("https://example.com/image.jpg")
    .placeholder(R.drawable.placeholder)
    .error(R.drawable.error)
    .resize(400, 400)
    .centerCrop()
    .into(imageView)
```

The `Picasso.get()` singleton is the common entry point. For custom configuration (a specific OkHttp client, a larger memory cache, a logging listener) build your own instance:

```kotlin
val picasso = Picasso.Builder(context)
    .downloader(OkHttp3Downloader(myOkHttpClient))
    .memoryCache(LruCache(50 * 1024 * 1024)) // 50 MB
    .build()
```

## Architecture / How It Works

The core object is `Picasso`, which owns a dispatcher, a memory cache, and a set of `RequestHandler`s. A `load()` call returns a `RequestCreator` — a mutable builder that accumulates options (`resize`, `centerCrop`, `rotate`, `transform`, `placeholder`, `error`, `tag`, `priority`) until a terminal call (`into`, `fetch`, or `get`) submits the request.

- **RequestHandlers** — pluggable resolvers keyed by URI scheme. Built-ins cover `http(s)`, `file`, `content`, `android.resource`, and asset URIs. Networking runs through a `Downloader`, and the default `OkHttp3Downloader` hands the actual fetch to OkHttp.
- **Memory cache** — an in-memory `LruCache` of decoded `Bitmap`s, sized by default to roughly 15% of available application RAM. This is Picasso's only cache that it manages itself.
- **Disk cache — delegated to HTTP.** Picasso does *not* maintain its own image disk cache. Persistent caching is OkHttp's HTTP response cache, which respects (and requires) standard `Cache-Control` / `Expires` headers from the origin. This is the single most misunderstood part of the library.
- **View lifecycle** — when a request targets an `ImageView`, Picasso attaches the request to the view via a `WeakReference`. If the view is recycled (as in a `RecyclerView`) and reused for a new `load()`, the in-flight request for the old URL is automatically cancelled. This is the feature that made Picasso pleasant to use in lists.
- **Threading** — decoding and transformation happen on a background thread pool; the final `Bitmap` is delivered on the main thread. `Transformation` implementations run off-main and must be thread-safe and side-effect-free.

The codebase was migrated to Kotlin around the 2.8 line (GitHub now reports Kotlin as the primary language), but the API shape and semantics are unchanged from the Java era. There are no coroutines and no Compose bindings; the model is imperative and `ImageView`-centric by design.

## Production Notes

**Disk caching depends on server headers.** Because persistent caching is OkHttp's HTTP cache, images served with `Cache-Control: no-store` (or no cache headers at all) will re-download every time and never persist across process restarts. Teams frequently report "Picasso isn't caching to disk" when the real cause is the origin's cache policy. Fixes: serve proper cache headers, or install an OkHttp client with a forced/rewriting cache interceptor. Glide, Fresco, and Coil all keep their own disk caches and do not have this footgun.

**No animated GIF support.** Picasso decodes a single frame. If you need GIFs (or WebP animation), Picasso is the wrong tool — this is a common reason teams historically reached for Glide or Fresco instead.

**No Jetpack Compose integration.** There is no `rememberPicassoPainter` equivalent. Using Picasso from Compose means bridging through `AndroidView` or a hand-rolled loader, which is awkward and is explicitly called out as a migration driver in the deprecation notice[^2].

**OutOfMemory on unbounded loads.** Loading full-resolution images straight into a view without `resize()` / `fit()` decodes the full bitmap into the memory cache. On image-heavy screens this is a classic OOM source. Always constrain dimensions to the target view; `fit()` defers sizing until the view is measured but cannot be combined with an explicit `resize()`.

**Global singleton pitfalls.** `Picasso.get()` uses a process-wide default instance and OkHttp client. If you need custom timeouts, interceptors (auth headers), or a shared OkHttp client with the rest of the app, you must build and thread your own `Picasso` instance — and either set it via `Picasso.setSingletonInstance()` early (it throws if called after `get()`) or inject it everywhere.

**Deprecation / maintenance risk.** No new Maven Central releases are planned. Bug fixes, new-API-level compatibility (edge-to-edge, predictive back, newer `targetSdk` behaviors), and security updates to transitive dependencies should not be expected. For a maintained library the on-call cost is low; for a deprecated one, plan a migration path (Coil's API is close enough that a wrapper layer is feasible).

## When to Use / When Not

**Use when:**
- You are maintaining an existing app already built on Picasso and a rewrite isn't justified yet.
- You need a tiny, predictable, `ImageView`-only loader for static images and control the image server's cache headers.
- You want minimal dependency weight and already depend on OkHttp.

**Avoid when:**
- You're starting a new project — use Coil (the officially recommended successor).
- Your UI is Jetpack Compose — Picasso has no idiomatic integration.
- You need animated GIF/WebP, aggressive bitmap pooling for very image-dense screens, or native-heap bitmap management (Fresco territory).
- You want an actively maintained dependency with ongoing releases.

## Alternatives

- coil-kt/coil — the officially recommended successor; Kotlin-coroutines-based, Compose-first, similar fluent API. Use this for any new Android project.
- bumptech/glide — feature-rich, animated GIF support, sophisticated bitmap pooling and transitions. Use when you need GIFs or heavier list/transform performance tuning.
- facebook/fresco — manages bitmaps in native/ashmem memory to reduce Java-heap pressure and OOMs. Use for very image-heavy apps targeting low-memory devices.
- square/okhttp — not an image loader, but the HTTP/caching layer Picasso sits on; relevant when you're really just fetching and caching bytes yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013-05 | Initial open-source release by Square[^1]. |
| 2.0 | 2014 | Rewrite; request-priority, `RequestHandler` extensibility. |
| 2.5.2 | 2015 | Long-lived, widely-adopted 2.x baseline. |
| 2.71828 | 2018-08 | Version named after Euler's number; OkHttp 3 default downloader, request handler refinements[^3]. |
| 2.8 | 2022 | Kotlin conversion; Java 8 + API 21 minimum; last public release[^1]. |
| — | 2024+ | Deprecated in README; migrate to Coil, no further Maven Central releases[^2]. |

## References

[^1]: Picasso README and project site. https://square.github.io/picasso/
[^2]: Picasso deprecation notice (repository README): "This library is deprecated. Please use alternatives like Coil for future projects... there will be no more public releases to Maven Central." https://github.com/square/picasso/blob/master/README.md
[^3]: Picasso releases. https://github.com/square/picasso/releases

## Tags

android, image-loading, image-cache, kotlin, java, okhttp, deprecated, mobile, ui, square
