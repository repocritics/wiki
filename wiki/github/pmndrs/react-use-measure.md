# pmndrs/react-use-measure

> A React hook that measures a DOM node's bounding rect and re-reports it on resize and scroll.

[GitHub repo](https://github.com/pmndrs/react-use-measure) ·
[npm](https://npmjs.com/react-use-measure) ·
[License: MIT](https://github.com/pmndrs/react-use-measure/blob/master/LICENSE)

## Overview

`react-use-measure` is a small single-purpose hook from the Poimandres (pmndrs) collective — the same group behind react-three-fiber, zustand, and react-spring. It returns a functional `ref` and a `bounds` object; attach the ref to any element and `bounds` gives you that element's `x`, `y`, `width`, `height`, `top`, `right`, `bottom`, and `left`. Unlike a one-shot `getBoundingClientRect()` call, the hook keeps the values current by subscribing to a `ResizeObserver` and, optionally, to scroll events on the element's ancestor scroll containers and the window[^1].

The problem it targets is narrow but real: `getBoundingClientRect()` returns coordinates relative to the viewport at the instant it is called, and it ignores the offsets contributed by nested scroll areas. There is no built-in reactive equivalent — you cannot ask the platform to notify you when an element's measured position changes[^2]. This hook wires up the observers and listeners so consuming components re-render with fresh geometry, which is what layout-driven UI (tooltips, popovers, drag targets, virtualized lists, canvas overlays, animated measurements) actually needs.

The defining tradeoff is scope discipline. The library does exactly one thing and does not try to be a general "element size + position + intersection" toolkit. That keeps it tiny and predictable, but it also means callers must understand `ResizeObserver` semantics — first-run zero values, layout-thrash costs of reading geometry, and the fact that `scroll: true` attaches listeners to every scrollable ancestor.

## Getting Started

```bash
npm install react-use-measure
# or: yarn add react-use-measure / pnpm add react-use-measure
```

```jsx
import useMeasure from 'react-use-measure'

function App() {
  const [ref, bounds] = useMeasure()

  // bounds is all-zero on the first render — the element hasn't been
  // measured yet. You get real numbers after layout, via a re-render.
  return (
    <div ref={ref}>
      {bounds.width} x {bounds.height}
    </div>
  )
}
```

With options — debounce measurements and track nested scroll offsets:

```jsx
const [ref, bounds] = useMeasure({
  debounce: { scroll: 50, resize: 0 },
  scroll: true,        // re-measure when ancestor scroll areas move
  offsetSize: true,    // use offsetWidth/Height to ignore parent CSS transforms
})
```

## Architecture / How It Works

The hook holds a `ResizeObserver` instance and a set of DOM event listeners, wired through a functional (callback) ref so it can run setup on mount and teardown on unmount without a `useEffect` cleanup race[^1].

- **Measurement source.** By default it calls `getBoundingClientRect()` on the observed node and copies the eight `RectReadOnly` fields into state. With `offsetSize: true` it substitutes `offsetWidth`/`offsetHeight` for `width`/`height`, which sidesteps the fact that `getBoundingClientRect()` returns the *transformed* box — useful when a parent applies `scale()` and you want the untransformed size.
- **Resize reactivity.** A `ResizeObserver` fires on any size change of the observed element. This is the primary trigger and covers the common "my box resized, update the number" case.
- **Scroll reactivity.** When `scroll: true`, the hook walks up the ancestor chain, finds scrollable containers, and attaches passive scroll listeners to each plus the window. Position fields (`top`, `left`, etc.) are relative to the viewport, so they only change on scroll — which is why scroll tracking is opt-in and off by default.
- **Debouncing.** `debounce` accepts either a single number or `{ scroll, resize }`, letting you throttle the two event sources independently. Scroll fires far more often than resize, so a scroll-only debounce is the typical configuration.
- **Ref identity.** The returned ref is the hook's own functional ref — it is not a mutable ref object you created. This is deliberate: unmount detection depends on the callback ref being invoked with `null`.

`RectReadOnly` is a plain readonly object, not a live `DOMRect`. Each update produces a new state value, so referential equality changes on every measurement — relevant if you feed `bounds` into a `useEffect`/`useMemo` dependency array.

## Production Notes

- **First render is always zero.** Every field is `0` until the element has been measured post-layout. Guard against dividing by `bounds.width`, and expect one extra render on mount. This surprises people building layout math that assumes synchronous measurement.
- **`ResizeObserver` availability.** The hook depends on `ResizeObserver`, which is present in all current browsers but absent in jsdom and older environments. In SSR the first client render matches the server's zeros, then hydrates and measures — usually fine, but content that shifts on measurement can cause a visible reflow. For tests or legacy targets, inject a polyfill via the `polyfill` option; the README recommends `@juggle/resize-observer`[^1].
- **`scroll: true` is not free.** It attaches listeners to *every* scrollable ancestor. On deeply nested layouts this is more work than people expect, and each scroll event triggers a `getBoundingClientRect()` read. Debounce scroll, and leave `scroll` off when the element's position is effectively static.
- **Layout thrash.** Reading `getBoundingClientRect()` forces the browser to flush pending layout. Measuring many elements per frame (e.g., one hook per row in a long list) can become a bottleneck; prefer measuring a container and computing offsets, or use a virtualization library that batches reads.
- **Transforms.** By default measured `width`/`height` reflect CSS transforms because `getBoundingClientRect()` does. If a parent scales the subtree and you want the intrinsic size, set `offsetSize: true`.
- **Bring your own ref.** Because the hook owns the ref, attaching your own ref to the same node requires a merge utility such as `react-merge-refs`[^1].
- **Maintenance cadence.** The library is small and stable rather than actively evolving; the last publish predates React 19's release, and there is a modest backlog of open issues. It works with React 18 and function components; there is no plan for feature growth, which for a utility this focused is a reasonable steady state rather than abandonment.

## When to Use / When Not

**Use when:**
- You need an element's live size *and* viewport position, kept current across resize and scroll.
- You're positioning tooltips, popovers, drag overlays, or canvas layers relative to a DOM node.
- You want a tiny dependency with a clear API instead of hand-wiring `ResizeObserver` and scroll listeners.

**Avoid when:**
- You only need size, never position — a bare `ResizeObserver` hook (e.g. `react-resizable-panels`' internals, or `use-resize-observer`) is lighter and does not touch scroll.
- You need to know when an element enters/leaves the viewport — that's `IntersectionObserver` territory, not this hook.
- You're measuring hundreds of nodes per frame — the per-node `getBoundingClientRect()` cost will dominate; measure containers instead.

## Alternatives

- ZeeCoder/use-resize-observer — thin `ResizeObserver` wrapper; use when you only need width/height and no scroll-aware position.
- streamich/react-use — the `useMeasure` in this grab-bag covers size only; use when you already depend on the library and don't need position tracking.
- juggle/resize-observer — a `ResizeObserver` polyfill, not a hook; use as the `polyfill` injection for old browsers or jsdom tests.
- floating-ui/floating-ui — full positioning engine for tooltips/popovers; use instead when the end goal is anchored floating UI, not raw measurements.
- react measurement via `getBoundingClientRect()` by hand — use when the measurement is one-shot (on click, on mount) and does not need to react to resize or scroll at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-10 | First release; ResizeObserver-based measurement hook[^1]. |
| 2.x | 2020–2021 | `scroll`, `debounce`, and `polyfill` options; TypeScript `RectReadOnly` types. |
| 2.1.x | 2022–2025 | `offsetSize` option; maintenance releases. Latest publish 2025-01[^3]. |

(Exact version-to-date mapping is not reconstructed here; consult the npm version history and git tags for precise release dates.)

## References

[^1]: pmndrs/react-use-measure README — usage, options, polyfill and multiple-ref notes. https://github.com/pmndrs/react-use-measure
[^2]: "Retrieve the position (X,Y) of an HTML element" — Stack Overflow discussion cited by the README on why no simple reactive API exists. https://stackoverflow.com/questions/442404/retrieve-the-position-x-y-of-an-html-element
[^3]: react-use-measure on npm — version history and current release. https://www.npmjs.com/package/react-use-measure

## Tags

react, hooks, typescript, resize-observer, dom-measurement, bounding-rect, layout, frontend, pmndrs, ui-utilities
