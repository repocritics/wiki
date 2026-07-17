# juliangarnier/anime

> A JavaScript animation engine that drives CSS properties, SVG, DOM attributes, and plain JS objects from one timeline API.

[GitHub repo](https://github.com/juliangarnier/anime) ·
[Official website](https://animejs.com) ·
[License: MIT](https://github.com/juliangarnier/anime/blob/master/LICENSE.md)

## Overview

Anime.js is a small, general-purpose animation library by Julian Garnier, first
released in 2016[^1]. It sits between hand-written `requestAnimationFrame` loops
and heavyweight timeline suites like GSAP: you describe target elements and the
property values to tween, and the library interpolates them on a shared internal
clock. It animates four things uniformly — CSS properties, individual CSS
transforms, SVG attributes/paths, DOM attributes, and JavaScript object
properties — which is its defining trait. There is no framework binding; it
manipulates whatever selector or object reference you hand it.

The project's history has two eras. V3 (the version most tutorials and Stack
Overflow answers still reference) exposed a single default `anime({ targets, ... })`
call and shipped as one bundle. V4, a ground-up rewrite released in 2025[^2],
replaced that with a tree-shakeable ES-module surface (`animate`, `createTimeline`,
`stagger`, `createTimer`, `createDraggable`, `onScroll`, and a `utils`/`svg`
namespace). The two APIs are not source-compatible; upgrading is a rewrite, not a
version bump, and the split is the single biggest source of confusion for anyone
arriving from an older codebase[^3].

The defining tension is scope. Anime.js is deliberately not a physics engine, not
a layout-animation system (no FLIP/shared-element helper), and not React-aware. It
is a timing-and-interpolation core with good ergonomics. That keeps it light, but
it means anything beyond "tween these values over this duration with this easing"
is your responsibility.

## Getting Started

```bash
npm install animejs
```

```javascript
// V4 — modular ESM API
import { animate, stagger } from 'animejs';

animate('.square', {
  x: 320,
  rotate: { from: -180 },
  duration: 1250,
  delay: stagger(65, { from: 'center' }),
  ease: 'inOutQuint',
  loop: true,
  alternate: true,
});
```

For a `<script>` tag without a bundler, the UMD/IIFE builds expose a global; V4
still ships CJS and UMD alongside ESM in `lib/`. V3 code using the default
`anime({ targets: '.square', translateX: 320 })` call will not run against a V4
install — pin `animejs@3` if you are not migrating.

## Architecture / How It Works

At the core is a single global engine: one `requestAnimationFrame` loop ticks all
active animations, timers, and timelines, rather than each animation owning its own
loop. Individual `animate()` calls create animation instances; a
`createTimeline()` composes instances with relative or absolute time offsets and
plays them against the same clock. This shared-ticker design is why staggering and
timeline sequencing stay in sync and why paused/seeked playback is deterministic.

Values are resolved per property. Anime.js reads the current computed value of a
target, parses units, and interpolates numerically toward the target value, then
writes it back each frame. Transforms (`x`, `rotate`, `scale`) are composed into a
single `transform` string so multiple transform tweens on one element don't clobber
each other — a common footgun in naive CSS animation. Easing accepts named
functions, cubic-bézier, steps, and spring parameters.

V4's module split is the architectural headline. Each capability — the timer, the
animation, the timeline, the draggable, the scroll observer, the scope, the SVG
path/morph helpers, the easing functions, `stagger`, and the `utils` — is a
separate import, so a bundler only ships what you use. V4 also exposes a WAAPI
path that hands eligible animations to the browser's native Web Animations API,
moving them off the main thread onto the compositor where the browser allows it.
Everything else runs as JS-driven interpolation on the main thread.

The library has no runtime dependencies and no virtual DOM. It is unaware of React
or Vue state — you animate refs/DOM nodes directly, and reconciling that with a
framework's render cycle (cleanup on unmount, re-running on state change) is left to
`createScope()` and your own effect hooks.

## Production Notes

- **V3 → V4 is a rewrite.** The default-export API is gone, property names and
  option keys changed (e.g. `translateX` → `x`, `easing` → `ease`,
  `direction: 'alternate'` → `alternate: true`), and the timeline API differs.
  Budget real migration time; the official v3→v4 guide is the reference[^3].
- **Main-thread cost.** JS-driven interpolation competes with your app's other
  work. Animating many elements, or expensive properties that trigger layout
  (`width`, `top`, `left`) rather than compositor-only ones (`transform`,
  `opacity`), will drop frames under load. Prefer transforms; reach for the WAAPI
  path when you need the browser to composite.
- **SVG paths need setup.** Line-drawing and morph effects depend on the SVG being
  structured correctly (matching point counts for morphs, accessible path length);
  these are the most support-question-heavy features.
- **No layout animations.** There is no built-in FLIP/shared-element transition.
  If you need list reordering or shared-element motion, that is a Motion/GSAP-Flip
  use case, not anime.js.
- **Framework cleanup is yours.** Without disposing animations on unmount you can
  leak timers or animate detached nodes. `createScope()` helps scope and revert
  animations, but you must wire it into the component lifecycle.
- **Funding model.** The project is MIT-licensed and free, sustained through GitHub
  Sponsors[^4]; V4's early access was offered to sponsors before the public
  release. Plan around a solo-maintainer cadence.

## When to Use / When Not

**Use when:**
- You want expressive tweening of CSS, SVG, and JS values with one small,
  dependency-free API.
- You need timelines, staggering, and easing without adopting a large suite.
- Bundle size matters and you want to tree-shake to only the features you call.
- You're doing SVG line-drawing, morphing, or decorative UI motion.

**Avoid when:**
- You need spring physics or gesture-driven, interruptible motion as a first-class
  model — a signals/physics library fits better.
- You need layout/shared-element transitions (FLIP) out of the box.
- You want deep React integration with declarative, state-driven animation.
- You're maintaining a V3 codebase and can't afford an API rewrite right now.

## Alternatives

- greensock/GSAP — use instead when you need the most complete timeline, plugin
  ecosystem (ScrollTrigger, MorphSVG, Flip), and battle-tested cross-browser edge
  cases, and can accept its packaging.
- motiondivision/motion — use instead in React or when you want WAAPI-first,
  gesture-aware, and layout (FLIP) animations declaratively.
- pmndrs/react-spring — use instead when you want physics/spring-based animation
  driven by React state rather than duration-based tweens.
- tweenjs/tween.js — use instead when you only need a tiny, generic value tweener
  (e.g. Three.js scenes) and don't need DOM/SVG conveniences.
- Web Animations API (native, no dependency) — use instead for simple keyframe
  animations you want composited off-thread without any library.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016 | Initial release as a single-file animation library[^1]. |
| 3.0 | 2019 | Default `anime()` API, timelines, SVG helpers; the long-lived era. |
| 3.2.x | 2020–2024 | Last of the V3 line; still widely pinned. |
| 4.0 | 2025 | Ground-up rewrite: tree-shakeable ESM, `animate`/`createTimeline`/`stagger`, WAAPI path, draggable & scroll modules[^2]. |

## References

[^1]: Anime.js repository and project site. https://animejs.com
[^2]: Anime.js V4 documentation. https://animejs.com/documentation
[^3]: "Migrating from v3 to v4", Anime.js wiki. https://github.com/juliangarnier/anime/wiki/Migrating-from-v3-to-v4
[^4]: Julian Garnier — GitHub Sponsors. https://github.com/sponsors/juliangarnier

## Tags

javascript, animation, animation-engine, svg, css, dom, tweening, timeline, frontend, ui-motion, esm, waapi
