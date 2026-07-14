# motiondivision/motion

> An animation library for React, JavaScript, and Vue built on a hybrid engine that mixes the native Web Animations API with its own JS runtime.

[GitHub repo](https://github.com/motiondivision/motion) ·
[Official website](https://motion.dev) ·
[License: MIT](https://github.com/motiondivision/motion/blob/main/LICENSE.md)

## Overview

Motion is the animation library formerly published as `framer-motion`. It was created by Matt Perry and grew up inside Framer as the declarative motion primitive for React, then merged with the separate framework-agnostic "Motion One" engine and was rebranded to plain "Motion" in late 2024[^1]. The React package now imports from `motion/react`; a vanilla-JS API ships from `motion`, and a Vue port ships as `motion-v`. The old `framer-motion` package name still resolves but new work lands under `motion`.

The library's defining idea is a *hybrid* engine: for CSS properties browsers can hardware-accelerate (transform, opacity, filter, clipPath) it hands the work to the native Web Animations API (WAAPI) so it runs off the main thread; for everything else — springs, interruptible animations, values you need to read mid-flight, layout transitions — it falls back to its own requestAnimationFrame loop[^2]. This split is why Motion can advertise "120fps GPU-accelerated" animation while still supporting physics and gesture-driven motion that WAAPI cannot express on its own.

The tension the project lives with is that this convenience sits on top of a lot of hidden machinery. The declarative `<motion.div animate={...}>` surface is easy; the moment you need exit animations, shared layout transitions, or tight bundle budgets, you are reasoning about `AnimatePresence` keys, FLIP reflows, and which feature bundle you imported. Motion is also a commercial project: a paid "Motion+" membership funds development and gates some premium APIs and tooling[^3], though the core library remains MIT.

## Getting Started

```bash
# React / JavaScript
npm install motion
# Vue
npm install motion-v
```

```jsx
// React — declarative component API
import { motion } from "motion/react"

function Card() {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ type: "spring", stiffness: 300, damping: 30 }}
    />
  )
}
```

```javascript
// Vanilla JS — imperative API
import { animate } from "motion"

animate("#box", { x: 100 }, { duration: 0.5 })
```

## Architecture / How It Works

Motion is layered. At the bottom is a framework-agnostic core that knows how to interpolate values, generate keyframes, and drive them either through WAAPI or a JS animation loop. On top sit the bindings: the React `motion` components, the imperative `animate()`/`scroll()`/`inView()` functions, and the Vue port.

- **Hybrid dispatch.** When a target property is WAAPI-acceleratable and the transition does not require JS features, Motion compiles it (including spring physics, which WAAPI has no native concept of, by baking the spring into a keyframe array) and hands it to the browser to run off-thread. Anything interruptible, physics-driven with live velocity, or requiring per-frame JS callbacks stays on the rAF loop[^2]. The upside is genuine off-main-thread smoothness; the cost is that WAAPI animations cannot be cheaply read or redirected mid-flight, so the two paths behave subtly differently.
- **MotionValues.** A `MotionValue` is a reactive value that updates the DOM directly, bypassing React re-renders. This is how Motion animates at 60–120fps without churning React's reconciler, and it is the mechanism behind `useScroll`, `useTransform`, and gesture springs.
- **Layout animations.** The `layout` prop uses the FLIP technique — measure First and Last positions, invert with a transform, then animate to identity. Shared-element transitions (`layoutId`) match a node across mount/unmount. FLIP animates transforms only (cheap), but the measurement pass forces synchronous layout reads and child elements need scale-distortion correction, which is where jank shows up at scale.
- **Presence.** `AnimatePresence` intercepts unmounts so exit animations can finish before React removes the node. It relies entirely on stable `key`s; mismatched keys are the single most common source of "my exit animation doesn't fire" reports.
- **Tree-shaking.** Importing the full `motion` component pulls in every feature. The `m` component plus `LazyMotion` with `domAnimation` or `domMax` feature bundles lets you load only what you use, which is the supported path for size-sensitive apps[^4].

## Production Notes

**Bundle size is a design decision, not a default.** The ergonomic `import { motion } from "motion/react"` drags in the full feature set. Teams that care about kilobytes must switch to `m` + `LazyMotion` and choose `domAnimation` (~15kb range) vs `domMax` (adds layout + drag). This is a real refactor, not a flag, so decide early.

**Layout animations are the biggest footgun.** They are genuinely useful and genuinely expensive: each `layout` element participates in a measure/invert/animate cycle that reads layout synchronously. Dozens of simultaneously-animating `layout` nodes, or `layout` combined with your own CSS `transform`, produce jank and surprising visual conflicts. Use them deliberately, not as a default on every list item.

**Server components.** Motion components are client components — they require `"use client"`. In Next.js App Router and other RSC setups you must keep animated subtrees on the client boundary; there is a `motion/react-client` entry to help, but you cannot animate inside a server component itself.

**The rebrand causes churn.** Migrating from `framer-motion` to `motion` is mostly a find-and-replace of the import path (`framer-motion` → `motion/react`), but mixing both packages in one tree can duplicate the runtime and break shared-layout matching. Pin one.

**Accelerated ≠ universal.** The "120fps GPU-accelerated" claim applies to WAAPI-eligible properties (transform/opacity/filter). Animating layout-affecting properties (width, height, top, margin) still runs on the main thread and still triggers reflow — Motion does not make those cheap. Prefer transform-based animation.

**Respect reduced motion.** `useReducedMotion` exists; honoring OS-level reduced-motion preferences is on you, not automatic.

## When to Use / When Not

**Use when:**
- You want a declarative React animation API with gestures, drag, springs, scroll-linking, and layout transitions in one library.
- You need shared-element / layout transitions without hand-rolling FLIP.
- You are already in React (or Vue via `motion-v`) and value ergonomics over minimal footprint.

**Avoid when:**
- You need timeline-heavy, framework-agnostic sequencing of complex scenes — GSAP is the stronger tool.
- Bundle size is your top constraint and you only need simple enter/leave — a CSS-first or drop-in tool is lighter.
- You are outside React/Vue/vanilla-DOM (e.g. React Native), where Motion's DOM-oriented engine does not apply.

## Alternatives

- pmndrs/react-spring — spring-physics animation with a hooks-first, more imperative model; use when you want fine-grained physical control and don't need layout transitions.
- greensock/GSAP — imperative, timeline-centric, framework-agnostic; use for complex choreographed sequences and non-React targets.
- juliangarnier/anime — small, imperative JS animation engine; use for lightweight standalone animation without a React binding.
- formkit/auto-animate — a single drop-in function for add/remove/move transitions; use when you want automatic list animation with near-zero API.
- pmndrs/react-three-fiber — use instead when your animation problem is actually 3D/WebGL rather than DOM.

## History

| Version | Date | Notes |
|---------|------|-------|
| framer-motion 1.0 | 2019 | Declarative React animation library by Matt Perry, successor to Pose[^1]. |
| Motion One | 2021 | Separate framework-agnostic WAAPI-based engine (motion.dev). |
| framer-motion 5 | 2021 | Shared layout animations via `layoutId` reworked. |
| framer-motion 6–11 | 2022–2024 | Hybrid WAAPI engine, `useScroll`, feature-bundle tree-shaking matured. |
| Rebrand to Motion | 2024 | `framer-motion` → `motion`; React import becomes `motion/react`[^1]. |
| motion-v | 2024–2025 | Official Vue port. |

## References

[^1]: Motion documentation, "Framer Motion is now Motion" / upgrade guide. https://motion.dev/docs/react-upgrade-guide
[^2]: Motion documentation, "Feature comparison" and hybrid-engine / performance notes. https://motion.dev/docs/react-motion-component
[^3]: Motion+ membership. https://motion.dev/plus
[^4]: Motion documentation, "Reduce bundle size" (`LazyMotion`, `m`, `domAnimation`/`domMax`). https://motion.dev/docs/react-reduce-bundle-size

## Tags

animation, react, vue, javascript, typescript, web-animations-api, spring-physics, gestures, layout-animation, frontend, ui
