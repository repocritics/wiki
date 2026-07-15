# animate-css/animate.css

> A drop-in stylesheet of ready-made CSS keyframe animations — add two class names, get a bounce or a fade, no JavaScript required.

[GitHub repo](https://github.com/animate-css/animate.css) ·
[Official website](https://animate.style/) ·
License: Hippocratic License (non-OSI, reported by GitHub as `NOASSERTION`)

## Overview

Animate.css is a single CSS file exposing a catalog of named keyframe
animations — `bounce`, `fadeIn`, `slideInLeft`, `zoomOut`, `flip`, and
dozens more — that you trigger by adding class names to an element. It was
created by Daniel Eden in 2011 and is one of the oldest and most-starred
front-end libraries on GitHub (~82.7k stars, ~16k forks). It ships no
JavaScript: an animation runs the moment its class is present in the DOM.

The library's whole value proposition is that it is *small and finished*.
There is no build step, no runtime, no configuration — you link a
stylesheet and reach for a class name. That also defines its ceiling:
because triggering is "class is present / class is absent," anything
beyond fire-once entrance/exit effects (scroll-linked reveals, sequenced
timelines, physics, interruptible transitions) requires you to add your
own JavaScript to toggle classes and listen for `animationend`. Animate.css
is a vocabulary of effects, not an animation engine.

Two facts dominate how the project should be read in 2026. First, **v4
(April 2020) was a breaking rewrite**: every class was namespaced with an
`animate__` prefix and a required `animate__animated` base class was
introduced, so v3 markup does not work on v4 without changes[^1]. Second,
the project is effectively **feature-complete and in low-activity
maintenance** — the last push to the default branch was mid-2024, and the
public release line has sat at v4.1.1 since 2020[^2]. This is stability,
not abandonment: a curated set of animations does not rot the way a
framework does. But do not expect new effects or active issue triage.

## Getting Started

```shell
npm install animate.css
# or: yarn add animate.css
```

Or via CDN, without a build step:

```html
<link rel="stylesheet"
  href="https://cdnjs.cloudflare.com/ajax/libs/animate.css/4.1.1/animate.min.css" />
```

Apply the base class plus one effect class (note the `animate__` prefix, v4+):

```html
<h1 class="animate__animated animate__bounce">Hello</h1>
```

Utility classes tune timing and repetition without custom CSS:

```html
<div class="animate__animated animate__fadeIn
            animate__delay-2s animate__slow animate__infinite">…</div>
```

You can also drive timing with CSS custom properties, per-element or globally:

```css
:root { --animate-duration: 800ms; --animate-delay: 0.5s; }
.my-element { --animate-duration: 2s; }
```

## Architecture / How It Works

There is almost no "architecture" — that is the point. The distributable
is a flat stylesheet containing `@keyframes` blocks and the classes that
bind them to `animation-name`, `animation-duration`, and
`animation-fill-mode`. The source is authored in modular files (one per
effect, grouped into `attention_seekers`, `bouncing_entrances`,
`fading_exits`, `sliding`, `zooming`, `specials`, etc.) and concatenated
into `animate.css` / `animate.min.css`.

Key design decisions worth understanding:

- **Class namespacing (v4).** Every animation lives under `animate__*` and
  requires the `animate__animated` base class, which sets
  `animation-duration` and `animation-fill-mode: both`. The prefix exists
  to avoid collisions with app CSS and utility frameworks — the reason for
  the v3→v4 break. A compatibility layer is documented for teams that
  cannot migrate markup immediately[^1].
- **CSS custom properties.** v4 exposes `--animate-duration`,
  `--animate-delay`, and `--animate-repeat` so global or per-element timing
  can be changed without overriding keyframes. Utility classes
  (`animate__slow`, `animate__delay-2s`, `animate__repeat-2`,
  `animate__infinite`) are thin wrappers over these variables.
- **No JavaScript, no state.** The library never observes the DOM. An
  effect plays when its class appears. To play an animation on scroll, on
  click, or in sequence, you add and remove classes yourself and listen for
  the `animationend` event — a pattern the docs spell out explicitly.
- **`prefers-reduced-motion` support.** The stylesheet wraps animations so
  that when the OS/browser reports reduced-motion, transitions are disabled
  automatically with no extra code[^3].

Because it is pure CSS, it composes cleanly with any framework (React, Vue,
Svelte, plain HTML) and any bundler — it is just a stylesheet import.

## Production Notes

- **Bundle weight.** The full minified stylesheet is on the order of tens
  of kilobytes and defines a large catalog you will mostly not use. Most
  effects on a page use a handful of animations; shipping the entire file
  is wasteful. Import only the source partials you need, or tree-shake/purge
  unused classes (PurgeCSS/Tailwind content scanning) in production builds.
- **`animation-fill-mode: both` retains final state.** The base class holds
  the element at its last keyframe. For exit animations (`fadeOut`, etc.)
  the element remains visually gone but still occupies layout and stays in
  the accessibility tree — you must remove it from the DOM (or set
  `display:none`) after `animationend` yourself.
- **The v3→v4 migration is real work.** Every class name changed. Codebases,
  CMS content, and third-party snippets referencing unprefixed classes
  (`class="animated bounce"`) silently do nothing on v4. Audit before
  upgrading; the pre-v4 docs are pinned to an old commit[^4].
- **Not an interaction system.** Scroll reveals, staggered lists, and
  interruptible/reversible motion are out of scope. Teams frequently pair
  Animate.css with a small scroll library (or write an IntersectionObserver)
  to add/remove classes — at which point evaluate whether a JS animation
  library would be simpler than the glue code.
- **License is the sharpest caveat.** Animate.css is under the **Hippocratic
  License**, an ethical-source license that restricts use by parties acting
  in violation of human-rights standards. It is **not OSI-approved and not
  recognized by SPDX** (hence GitHub's `NOASSERTION`)[^5]. Many corporate
  open-source-compliance processes reject or flag non-OSI licenses; legal
  review is warranted before shipping it in a commercial product, even
  though the code itself is trivially replaceable.

## When to Use / When Not

**Use when:**
- You want a few tasteful entrance/exit/attention effects with zero setup.
- Prototypes, marketing pages, docs, and CMS themes where a stylesheet link
  is the whole integration budget.
- You want animations that respect `prefers-reduced-motion` for free.

**Avoid when:**
- You need scroll-triggered, sequenced, physics-based, or interruptible
  motion — that is an animation engine's job, not a class list's.
- Bundle size is tightly controlled and you can hand-write the two or three
  keyframes you actually use.
- Your organization's license policy forbids non-OSI / ethical-source
  licenses.

## Alternatives

- animejs/anime.js — lightweight JS engine; use when you need timelines,
  staggering, and programmatic control rather than fixed class-triggered effects.
- greensock/GSAP — the professional standard for complex, sequenced,
  performant web animation; use when motion is a core product feature.
- michalsnik/aos — Animate On Scroll; use when you specifically want reveal-on-scroll without writing observer glue.
- IanLunn/Hover.css — a comparable pure-CSS catalog focused on hover/interaction effects rather than entrances.
- framer/motion — declarative animation for React with gestures and layout
  transitions; use when you are already in React and want stateful motion.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2011-10 | Created by Daniel Eden; unprefixed classes (`animated`, `bounce`). |
| 3.x | through ~2019 | Long-lived stable line; last of the unprefixed API[^4]. |
| 4.0.0 | 2020-04 | Breaking rewrite: `animate__` prefix, required `animate__animated` base class, CSS custom properties for timing[^1]. |
| 4.1.1 | 2020 | Current release; project moves into low-activity maintenance[^2]. |

## References

[^1]: Animate.css v4 migration guide / prefix documentation. https://animate.style/
[^2]: GitHub releases and repository activity (last push mid-2024; latest tag v4.1.1). https://github.com/animate-css/animate.css/releases
[^3]: WebKit, "Responsive Design for Motion" — `prefers-reduced-motion`. https://webkit.org/blog/7551/responsive-design-for-motion/
[^4]: Pre-v4 (v3.x and under) documentation, pinned to an old commit. https://github.com/animate-css/animate.css/tree/a8d92e585b1b302f7749809c3308d5e381f9cb17
[^5]: Hippocratic License. https://firstdonoharm.dev/

## Tags

css, css-animations, keyframes, stylesheet, frontend, ui, web-animation, no-javascript, hippocratic-license, cdn
