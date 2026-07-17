# pmndrs/use-gesture

> Utility that turns raw mouse, touch, wheel, and scroll events into normalized gesture state for React and vanilla JavaScript.

[GitHub repo](https://github.com/pmndrs/use-gesture) ·
[Official website](https://use-gesture.netlify.app) ·
[License: MIT](https://github.com/pmndrs/use-gesture/blob/main/LICENSE)

## Overview

use-gesture is a small library that binds richer pointer events to a DOM element and hands you a continuously-updated gesture state: movement deltas, absolute offset, velocity, direction, distance, whether the gesture is active, and more. It does not move or animate anything itself — you read the state and feed it into an animation library (most commonly react-spring, also from the pmndrs/poimandres collective, but framer-motion or any imperative style setter works). This separation is the whole design: use-gesture is the input layer, the animation library is the output layer[^1].

The React binding is the headline surface (`@use-gesture/react`), exposing hooks — `useDrag`, `useMove`, `useHover`, `useScroll`, `useWheel`, `usePinch`, and the combined `useGesture`. Each returns a `bind` function whose spread output (`{...bind()}`) attaches the necessary listeners to a component. A framework-agnostic core (`@use-gesture/core`) does the actual event math, and a separate `@use-gesture/vanilla` package exposes class-based recognizers (`DragGesture`, `PinchGesture`, …) for non-React code.

The defining tension is that use-gesture solves a genuinely fiddly problem — cross-device pointer normalization, kinetic drag, pinch-zoom, rubberbanding against bounds — but leaves the visual result entirely to you. That makes it flexible and unopinionated, but means "make a draggable card" is only half a solution out of the box; the other half is whatever animates the transform. It is also, as of mid-2026, in a low-activity maintenance posture: the last published commit to the default branch was July 2024[^2], so treat it as stable-and-settled rather than actively evolving.

## Getting Started

```bash
npm install @use-gesture/react
# vanilla JS instead:  npm install @use-gesture/vanilla
```

```jsx
import { useSpring, animated } from '@react-spring/web'
import { useDrag } from '@use-gesture/react'

function Draggable() {
  const [{ x, y }, api] = useSpring(() => ({ x: 0, y: 0 }))

  // Handler fires on every pointer event during the gesture.
  const bind = useDrag(({ down, movement: [mx, my] }) => {
    api.start({ x: down ? mx : 0, y: down ? my : 0 })
  })

  // touchAction: 'none' is REQUIRED to stop the browser from
  // scroll-hijacking the drag on touch devices.
  return <animated.div {...bind()} style={{ x, y, touchAction: 'none' }} />
}
```

Vanilla equivalent (`new DragGesture(el, handler)` returns an instance with a `.destroy()` you must call to detach listeners).

## Architecture / How It Works

The core (`@use-gesture/core`) is framework-neutral. Each gesture is a recognizer that subscribes to browser events — by default Pointer Events, with fallbacks — and maintains a state object. On each event it computes derived values: `movement` (delta since gesture start), `offset` (accumulated across gestures), `velocity`, `direction`, `distance`, `elapsedTime`, and flags like `first`, `last`, `active`, `tap`, `swipe`. Your handler is called with that shared state on every tick.

The React layer is thin: hooks build a config, register the recognizers, and return a `bind` function. Spreading `bind()` onto an element wires up listeners. Critically, for gestures that need `preventDefault` (drag on touch, wheel, pinch), use-gesture attaches native non-passive listeners via `addEventListener` on the target rather than relying on React's synthetic event system — React attaches many listeners as passive at the root, which cannot cancel default scrolling. This is why some options (`eventOptions.passive: false`) and the mandatory `touch-action: none` exist.

Behavior is driven by config rather than imperative code. Common options: `bounds` (clamp movement to a box), `rubberband` (elastic resistance past bounds), `from` (seed the offset), `axis` / `lockDirection`, `threshold`, `filterTaps`, `pointer: { touch: true }`, and `transform`. `usePinch` reports `da` (distance/angle) and `origin`, enabling zoom-and-rotate. `useGesture` composes several recognizers with a shared config and per-gesture overrides.

Because it only reads events and returns numbers, use-gesture holds no visual state and is animation-library-agnostic. That is the coupling story: it couples tightly to the DOM event model and to whatever you do with the numbers, but not to any renderer.

## Production Notes

- **touch-action is the number-one footgun.** Any draggable element needs `touch-action: none` (or `pan-y`/`pan-x` for single-axis) in CSS or inline style. Without it, mobile browsers claim the gesture for native scrolling and drags stutter or die. The docs warn about this repeatedly; it is still the most common bug report pattern.
- **You must call `bind()`, and spread the result.** `{...bind()}` — forgetting the call or the spread silently attaches nothing. With `useGesture`/multiple targets you can pass arguments to `bind(...)` to key handlers.
- **Passive listeners and `preventDefault`.** To cancel default browser behavior (page zoom on pinch, scroll on wheel) you need non-passive listeners: set `eventOptions: { passive: false }`. Some browsers still refuse to let you cancel certain wheel/touch defaults on the document.
- **Maintenance status.** Low commit activity and no release in a long stretch as of 2026; ~50 open issues[^2]. It is widely depended on and API-stable, but do not expect fixes for new browser quirks quickly. Pin versions and test on target devices rather than assuming upstream will chase regressions.
- **SSR is safe** — no `window` access at import time; listeners attach in effects. Works under React 18 StrictMode's double-invoked effects because binding is ref-based and idempotent.
- **Upgrade pain lives at the v9→v10 boundary** (see History): the package was renamed and the API changed shape. Anything still on `react-use-gesture` is on the pre-rewrite line and needs a real migration, not a version bump.
- **`offset` vs `movement` confusion.** `movement` resets to `[0,0]` at the start of each gesture; `offset` accumulates across gestures. Persisting a card's position across multiple drags means reading `offset` (usually seeded with `from`), not `movement` — mixing them up produces jumps on the second drag.
- **Bounds and rubberband are computed, not enforced on the element.** use-gesture clamps the numbers it reports; it does not constrain the DOM node. If your animation setter ignores the clamped value, the element still moves out of bounds. The library reports; you apply.

## When to Use / When Not

**Use when:**
- You need drag / pinch / wheel / scroll gestures with correct cross-device pointer handling and don't want to hand-roll velocity, bounds, and rubberbanding.
- You already use react-spring or another animation setter and want a clean input→output split.
- You want a tiny, dependency-light gesture layer that works in React or vanilla.

**Avoid when:**
- You want gestures and animation bundled together with less wiring — framer-motion's `drag` prop or `@react-spring` + a higher-level kit gets you there faster.
- You need full multi-touch, physics-based, or canvas/WebGL gesture handling beyond DOM pointer events.
- You want an actively-evolving dependency with responsive maintenance; this one is in maintenance mode.

## Alternatives

- framer/motion — use instead when you want drag/pan gestures and animation in one library via the `drag` prop, with less manual state wiring.
- pmndrs/react-spring — pairs with use-gesture rather than replacing it; use its `useSpring`/`useTransition` as the animation output layer.
- react-grid-layout/react-draggable — use instead when you only need basic element dragging and want a ready-made component, not a gesture primitive.
- atlassian/react-beautiful-dnd (now deprecated) or clauderic/dnd-kit — use dnd-kit instead when the real task is sortable lists / drag-and-drop with accessibility, not free-form gestures.
- xnimorz/use-debounce is unrelated; for raw pointer math without React, use `@use-gesture/vanilla` from this same repo.

## History

| Version | Date | Notes |
|---------|------|-------|
| react-use-gesture 1.x–9.x | 2018–2020 | Original single-package React library; event handling built on touch/mouse events. |
| v10.0 | 2021 | Major rewrite. Split into `@use-gesture/core` + `@use-gesture/react` + `@use-gesture/vanilla` monorepo, full TypeScript, Pointer Events by default, revised config-driven API. Renamed from `react-use-gesture`.[^3] |
| v10.2–10.3 | 2022–2024 | Incremental fixes and option refinements on the v10 line; last default-branch commit July 2024.[^2] |

## References

[^1]: use-gesture README and documentation — installation, hooks table, and the "combine with an animation library" guidance. https://use-gesture.netlify.app
[^2]: GitHub API metadata for pmndrs/use-gesture — last push 2024-07-15, ~53 open issues, 9.6k stars, MIT license (retrieved 2026-07). https://github.com/pmndrs/use-gesture
[^3]: use-gesture documentation, migration notes for the v10 rewrite (package rename `react-use-gesture` → `@use-gesture/react`, monorepo split). https://use-gesture.netlify.app/docs
## Tags

react, gestures, drag, pinch, pointer-events, touch, typescript, animation, hooks, frontend, pmndrs
