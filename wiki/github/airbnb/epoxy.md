# airbnb/epoxy

> Annotation-driven view models for building heterogeneous RecyclerView screens on Android, extracted from Airbnb's production app.

[GitHub repo](https://github.com/airbnb/epoxy) ·
[Wiki / docs](https://github.com/airbnb/epoxy/wiki) ·
[License: Apache-2.0](https://github.com/airbnb/epoxy/blob/master/LICENSE)

## Overview

Epoxy is an Android library that turns a `RecyclerView` full of mixed view
types into a declarative list. You describe the screen in a `buildModels`
method — "a header, then N photos, then a loader" — and Epoxy computes the
diff against the previous state and animates the adapter to match. It removes
the hand-written `Adapter`, `ViewHolder`, view-type integer, and `notifyItem*`
bookkeeping that a multi-type list normally requires.

Its defining mechanism is annotation processing. You annotate a custom view
(`@ModelView`), a databinding layout (`@EpoxyDataBindingLayouts`), or a
ViewHolder model (`@EpoxyModelClass`), and a code generator emits a
`*Model_` builder class plus a Kotlin DSL extension. This is the central
tradeoff: the developer experience of the generated DSL is bought with a
compile-time processor in the build graph. Under KAPT that processor is a
common contributor to slow Android builds; the library now recommends KSP,
which is faster but does not support databinding models[^1].

Epoxy was developed at Airbnb to back most of the main screens in its Android
app and open-sourced in 2016[^2]. It remains maintained but slowly: releases
have become infrequent, and the pattern it embodies — imperative View-based
lists — has largely been superseded by Jetpack Compose's `LazyColumn` for
greenfield code. Epoxy is best understood today as the mature answer for large
existing View/XML codebases, not the default choice for a new app.

## Getting Started

```groovy
// module build.gradle — KSP path (recommended)
plugins {
  id 'com.android.application'
  id 'kotlin-android'
  id 'com.google.devtools.ksp'
}

dependencies {
  implementation "com.airbnb.android:epoxy:$epoxyVersion"
  ksp           "com.airbnb.android:epoxy-processor:$epoxyVersion"
}
```

```kotlin
// A custom view becomes a model via annotations
@ModelView(autoLayout = Size.MATCH_WIDTH_WRAP_HEIGHT)
class HeaderView(context: Context) : LinearLayout(context) {
  @TextProp fun setTitle(text: CharSequence) { titleView.text = text }
}

// Declare the list directly on an EpoxyRecyclerView — no controller class
epoxyRecyclerView.withModels {
  header {
    id("header")            // stable id is REQUIRED for every model
    title("My Photos")
  }
  photos.forEach { p ->
    photoView { id(p.id); url(p.url) }
  }
  if (loadingMore) loaderView { id("loader") }
}
```

## Architecture / How It Works

Epoxy has three layers. The **annotation processor** (`epoxy-processor`, run
via KAPT or KSP) reads `@ModelView`/`@EpoxyModelClass`/`@EpoxyAttribute` and
generates immutable `*Model_` builder subclasses plus Kotlin DSL functions.
The **runtime** (`EpoxyController` / `EpoxyAdapter`) holds the current model
list. The **diffing engine** compares the old and new model lists and emits
granular `notifyItem*` calls to the RecyclerView.

The control flow is: your data changes, you call `requestModelBuild()`,
`buildModels()` runs and produces a new immutable model list, and Epoxy diffs
it against the previous list. Diffing runs on a background thread; the
resulting dispatch to the adapter happens on the main thread. Equality of two
models drives the diff — Epoxy compares each model's generated `hashCode`
(built from its `@EpoxyAttribute` fields) to decide whether an item is
unchanged, moved, or needs a partial rebind. Because diffing keys on model
`id()`, **every model must be given a stable, unique id** or the controller
throws at build time.

`TypedEpoxyController` / `Typed2EpoxyController` add a typed `setData(...)`
that calls `requestModelBuild()` for you. `EpoxyRecyclerView` is a
`RecyclerView` subclass that wires the adapter, handles model rebuilds, and
adds view recycling across screens via a shared view pool. Airbnb's own state
library, Mavericks (formerly MvRx), pairs with Epoxy so that a state change
triggers a model rebuild automatically.

The coupling that matters: Epoxy sits on top of the standard AndroidX
`RecyclerView` and `DiffUtil` primitives — it orchestrates rather than
replaces them. That keeps it compatible with existing `LayoutManager`, item
decorations, and span-size logic, but also means it inherits RecyclerView's
constraints and the cost of large binds on the main thread.

## Production Notes

- **Build time is the recurring complaint.** The annotation processor runs on
  every build touching annotated views. On KAPT this is measurable on large
  modules; migrating to KSP is the primary mitigation, but only for projects
  that do not depend on Epoxy's databinding integration, which KSP cannot
  support because Android databinding itself is KAPT-based[^1].
- **Missing/duplicate ids are the classic footgun.** A model without `id()`,
  or two models sharing an id, throws `IllegalStateException` at build time.
  Ids derived from list position (rather than stable data ids) defeat diffing
  and cause full rebinds and lost scroll/animation state.
- **`buildModels` runs on the main thread.** Only the diff is backgrounded.
  Building thousands of models, or doing formatting/allocation inside
  `buildModels`, blocks the UI thread; keep model construction cheap and move
  work into the data layer.
- **Generated classes and IDE friction.** `*Model_` types and DSL functions
  only exist after a successful build; a red IDE and "unresolved reference"
  errors on a clean checkout usually mean the processor hasn't run yet.
- **Release cadence has slowed.** The 5.1.4 → 5.2.0 gap spanned nearly two
  years (Jan 2024 → Nov 2025)[^3]. Still receiving fixes, but treat it as
  maintenance-mode; Epoxy predates Compose and offers no first-party bridge,
  so teams migrate whole screens rather than mixing at the item level.

## When to Use / When Not

**Use when:**
- You maintain a large View/XML Android app with complex, heterogeneous
  RecyclerView screens and want to delete adapter boilerplate.
- You already use Airbnb's Mavericks or a similar unidirectional-state setup.
- You need automatic, animated diffing across many view types without hand-
  writing `DiffUtil.ItemCallback`.

**Avoid when:**
- You are starting a new app — Jetpack Compose `LazyColumn`/`LazyRow` covers
  the same ground without a code generator.
- Your list is a single homogeneous view type — a plain `ListAdapter` +
  `DiffUtil` is simpler and adds no build-time processor.
- Build times are already a pain point and you cannot move off KAPT (e.g. you
  rely on Epoxy databinding models).

## Alternatives

- androidx/androidx `ListAdapter` + DiffUtil — first-party, no code generation; use when your list is simple or single-type and you don't want a processor.
- androidx/androidx Compose `LazyColumn` — use when the app is (or is going) Compose-first and you want lists without a code generator.
- lisawray/groupie — lighter multi-type RecyclerView grouping without annotation processing; use when you want Epoxy's ergonomics with less build cost.
- airbnb/mavericks — not a list library but the state layer Epoxy is designed to pair with; use alongside Epoxy, not instead of it.
- Square's cashapp/molecule — build UI state with Compose runtime feeding legacy Views; use when bridging Compose state into an existing View list.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-08 | Open-sourced from Airbnb's Android app[^2]. |
| 4.0.0 | 2020-09-08 | AndroidX-based line[^3]. |
| 5.0.0 | 2022-09-29 | KSP support for the annotation processor[^3]. |
| 5.1.4 | 2024-01-25 | Last release before a ~22-month gap[^3]. |
| 5.2.0 | 2025-11-22 | Resumed releases after long pause[^3]. |
| 5.2.1 | 2026-01-23 | Latest release as of this writing[^3]. |

## References

[^1]: Epoxy README — KSP is recommended over KAPT for speed, but "DataBinding models are not supported with KSP, as Android's DataBinding library itself uses KAPT." https://github.com/airbnb/epoxy#readme
[^2]: "Epoxy: Airbnb's View Architecture on Android," Airbnb Engineering. https://medium.com/airbnb-engineering/epoxy-airbnbs-view-architecture-on-android-c3e1af150394
[^3]: Epoxy releases page (dates verified against the GitHub Releases API). https://github.com/airbnb/epoxy/releases

## Tags

android, java, kotlin, recyclerview, ui, annotation-processing, ksp, kapt, list, airbnb, view-architecture, databinding
