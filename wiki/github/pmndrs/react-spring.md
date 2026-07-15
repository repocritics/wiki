# pmndrs/react-spring

> A spring-physics animation library for React that writes values straight to the DOM, skipping React re-renders on every frame.

[GitHub repo](https://github.com/pmndrs/react-spring) ·
[Official website](https://www.react-spring.dev/) ·
[License: MIT](https://github.com/pmndrs/react-spring/blob/next/LICENSE)

## Overview

react-spring is an animation library built on spring physics rather than
fixed-duration timelines. Instead of "animate opacity from 0 to 1 over 300ms",
you describe a target and a spring configuration (tension, friction, mass), and
the library integrates the motion frame by frame until it settles. This makes
interruptions and mid-flight target changes behave naturally — a spring already
in motion just re-aims at the new value instead of restarting an easing curve.
It is maintained under the pmndrs (poimandres) collective, the same group behind
react-three-fiber, zustand, and jotai, and was created by Paul Henschel[^1].

The library's defining architectural choice is that animated values do not flow
through React state. Committing every frame to a `setState` would trigger a
reconcile on each tick; instead react-spring hands you `animated.*` components
that subscribe to a raw value and mutate the underlying DOM node (or the
react-three-fiber object) directly. React renders the component once; the
animation loop takes over from there. This is the same performance strategy
Framer Motion uses.

The central tension is ergonomics versus surprise. Because animated values live
outside React, you must wrap the element in `animated`, and you cannot read the
live value as a plain number in your render body — it is an opaque `SpringValue`.
The v9 rewrite (2021) split the library into per-target packages and moved fully
to TypeScript, the last major migration most existing codebases had to absorb[^2].

## Getting Started

Install the package for your render target, not a monolithic `react-spring`:

```shell
npm install @react-spring/web      # react-dom
# npm install @react-spring/three  # react-three-fiber
# npm install @react-spring/native # react-native
```

```jsx
import { animated, useSpring } from '@react-spring/web'

function FadeIn({ isVisible, children }) {
  const styles = useSpring({
    opacity: isVisible ? 1 : 0,
    y: isVisible ? 0 : 24,        // shorthand for transform: translateY
    config: { tension: 210, friction: 20 },
  })

  return <animated.div style={styles}>{children}</animated.div>
}
```

The `animated.div` is required — a plain `<div>` cannot consume a `SpringValue`.
For derived values use `.to()`: `styles.opacity.to(o => o * 100)` produces a new
animated value without re-rendering.

## Architecture / How It Works

The monorepo is layered. `@react-spring/rafz` is a small requestAnimationFrame
scheduler; `@react-spring/core` holds `SpringValue`, `Controller`, and the hooks;
`@react-spring/animated` provides the `animated` factory; and target packages
(`web`, `three`, `native`, `konva`, `zdog`) wire the factory to a renderer[^3].
This is why you import from `@react-spring/web` and not a top-level entry — the
animated components are host-specific.

A `SpringValue` is a single animatable number/string that owns its own physics
state. A `Controller` groups several `SpringValue`s (one spring per animated
property). Hooks are thin wrappers over these:

- **`useSpring`** — one controller, one set of props.
- **`useSprings`** — N controllers driven by an array/function.
- **`useTrail`** — N springs that chase each other with a stagger.
- **`useTransition`** — mount/unmount animation with enter/leave/update phases;
  the hard case the library exists to solve, since leaving elements must persist
  in the tree until their spring settles.
- **`useChain`** — sequences multiple refs with relative timing.

Springs are the default, but a `config` can specify `{ duration, easing }` to
fall back to time-based tweening when physics is the wrong model (e.g. a precise
loading bar). Interpolation via `.to()` runs on the animation thread and never
touches React. The imperative API (`useSpring(() => ({...}))` returning a
`SpringRef`) lets you call `api.start()`, `api.pause()`, `api.stop()` outside the
render cycle, which is how you drive animations from event handlers or gestures
(commonly paired with @use-gesture/react from the same collective).

## Production Notes

- **The `animated` wrapper is non-negotiable and easy to forget.** Passing a
  `SpringValue` to a plain element renders `[object Object]` or silently does
  nothing. Third-party components need `animated(MyComponent)` and must forward
  `style`/`ref`.
- **Transforms are composed, units are not inferred.** `x`, `y`, `scale`,
  `rotate` shorthands map to `transform`; mixing them with a raw `transform`
  string in the same spring conflicts. Numeric values that need units (e.g.
  `width: '50%'`) must be interpolated to strings via `.to()`.
- **`useTransition` keys and item identity are the usual bug source.** If the
  key function is unstable, elements re-mount instead of animating, and leaving
  items can pile up. Read the item-vs-key semantics carefully before using it in
  lists.
- **StrictMode double-invoke.** Effects and refs run twice in development
  StrictMode; imperative `SpringRef` setups that assume single invocation can
  misbehave in dev while being fine in production.
- **SSR.** Springs start on the client; server-rendered markup reflects the
  `from` state, so animating layout-affecting properties on first paint can cause
  a visible jump unless initial and rendered states match.
- **v8 → v9 was a breaking migration**: scoped packages, TypeScript-first types,
  and a reworked imperative API. Codebases still on the old `react-spring` import
  path predate this and should budget time to move[^2].

## When to Use / When Not

**Use when:**
- You want interruptible, physics-based motion that re-targets gracefully
  (drag-and-drop, gestures, springy UI).
- You are animating react-three-fiber scenes and want the same API on the web.
- You need enter/leave transitions on mounting/unmounting lists.
- You want animations that don't re-render React on every frame.

**Avoid when:**
- The effect is a simple one-shot fade or hover — CSS transitions are smaller and
  need no wrapper components.
- You want a large prebuilt gesture/layout-animation toolkit with `layout` props
  and variants out of the box — Framer Motion covers more of that surface.
- Your team finds the "values live outside React" model confusing and you don't
  need the performance; a `setState`-based library may be easier to reason about.

## Alternatives

- framer-motion (motion/react) — declarative `motion.*` components with layout
  animations, variants, and gestures built in; reach for it when you want batteries
  included over physics primitives.
- greensock/GSAP — imperative, timeline-first, framework-agnostic; use when you
  need precise choreography and scrubbing rather than springs.
- chenglou/react-motion — the spiritual predecessor that popularized spring-based
  React animation; effectively unmaintained, use react-spring instead.
- reactjs/react-transition-group — minimal mount/unmount lifecycle hooks with no
  animation engine; use when you only need CSS-class transition timing.
- formkit/auto-animate — one-line automatic layout transitions; use when you want
  zero-config list/DOM-change animation and nothing more.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2018-03 | Repo created; spring-physics React animation, inspired by react-motion[^1]. |
| 8.0 | 2019 | Hooks-based API (`useSpring`, `useTransition`) as the primary interface. |
| 9.0 | 2021 | Full TypeScript rewrite; split into scoped `@react-spring/*` target packages; reworked imperative `SpringRef` API[^2]. |
| next | ongoing | Active development continues on the `next` default branch (last push 2026-07)[^4]. |

At ~29k stars and 1.2k forks with commits within the last day, the project is
actively maintained rather than in maintenance-only mode[^4].

## References

[^1]: react-spring, pmndrs collective; created by Paul Henschel (drcmda). https://github.com/pmndrs/react-spring
[^2]: react-spring documentation, migration and package structure. https://www.react-spring.dev/docs/getting-started
[^3]: Scoped npm packages: @react-spring/web, @react-spring/three, @react-spring/native, @react-spring/konva, @react-spring/zdog, core, animated, shared, rafz. https://www.npmjs.com/org/react-spring
[^4]: GitHub repository metadata, retrieved 2026-07 (29,127 stars, 1,219 forks, MIT, default branch `next`, last push 2026-07-13). https://github.com/pmndrs/react-spring

## Tags

react, animation, spring-physics, typescript, react-three-fiber, frontend, ui, hooks, gestures, javascript
