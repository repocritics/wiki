# floating-ui/floating-ui

> Low-level positioning engine for tooltips, popovers, and dropdowns — the successor to Popper.js.

[GitHub repo](https://github.com/floating-ui/floating-ui) ·
[Official website](https://floating-ui.com) ·
[License: MIT](https://github.com/floating-ui/floating-ui/blob/master/LICENSE)

## Overview

Floating UI positions "floating" elements — tooltips, popovers, dropdowns, menus — relative to an anchor (usually a button), while keeping them inside the viewport when space runs out. It solves two problems that look trivial and are not: computing an `(x, y)` for a floating element next to a reference, and adjusting that position when the element would otherwise collide with a scroll container or the viewport edge[^1].

The project is the direct continuation of **Popper.js**. Popper 1.x (2016) and the TypeScript rewrite Popper 2.x (2020) live on the `v2.x` branch of this same repo; the library was renamed Floating UI around 2021–2022 and re-architected around a small `computePosition` core plus composable middleware[^2]. If you have seen `data-popper-*` attributes or `createPopper()` in a codebase, you are looking at the ancestor of this library.

The defining design choice is that Floating UI is a **positioning primitive, not a component library**. It gives you coordinates and a middleware pipeline; it does not give you a styled Tooltip component. On the vanilla and Vue sides it is purely positioning. On the React side (`@floating-ui/react`) it additionally ships interaction and accessibility hooks — hover/click/focus/dismiss handling, focus management, ARIA roles, list navigation — because those are the hard parts of building an *accessible* floating element, not the positioning. This split is the main thing to understand before adopting it: you are assembling behavior, not importing a finished widget.

## Getting Started

Install the package for your platform (they share the `@floating-ui/core` engine):

```shell
npm install @floating-ui/dom     # vanilla DOM
npm install @floating-ui/react   # React DOM + interactions
npm install @floating-ui/vue     # Vue
```

Vanilla DOM, positioning a tooltip above a button with collision handling:

```js
import { computePosition, offset, flip, shift, autoUpdate } from '@floating-ui/dom';

// Recompute on scroll, resize, and layout shifts; returns a cleanup fn.
const cleanup = autoUpdate(button, tooltip, () => {
  computePosition(button, tooltip, {
    placement: 'top',
    middleware: [offset(6), flip(), shift({ padding: 5 })],
  }).then(({ x, y }) => {
    Object.assign(tooltip.style, { left: `${x}px`, top: `${y}px` });
  });
});
```

React, using the hook and an interaction:

```jsx
import { useFloating, useHover, useInteractions, offset, flip, shift } from '@floating-ui/react';

function Tooltip() {
  const [open, setOpen] = React.useState(false);
  const { refs, floatingStyles, context } = useFloating({
    open, onOpenChange: setOpen,
    middleware: [offset(6), flip(), shift()],
  });
  const hover = useHover(context);
  const { getReferenceProps, getFloatingProps } = useInteractions([hover]);

  return (
    <>
      <button ref={refs.setReference} {...getReferenceProps()}>Hover me</button>
      {open && (
        <div ref={refs.setFloating} style={floatingStyles} {...getFloatingProps()}>
          Tooltip
        </div>
      )}
    </>
  );
}
```

## Architecture / How It Works

Two layers:

1. **`@floating-ui/core`** — platform-agnostic geometry. It knows nothing about the DOM. It works entirely against a `Platform` interface (get element rects, get dimensions, get offset parent). This is what lets the same engine drive React Native or a Canvas/WebGL renderer[^3].
2. **`@floating-ui/dom`** — implements that `Platform` for the browser (reading `getBoundingClientRect`, walking offset parents, handling scroll). `@floating-ui/react`, `@floating-ui/react-dom`, and `@floating-ui/vue` are framework bindings on top of `dom`.

The core primitive is **`computePosition(reference, floating, options)`**, which returns `{ x, y, placement, middlewareData }`. It is one-shot: it computes a position *once*. It does not watch for changes. Keeping a floating element attached during scroll/resize is the job of **`autoUpdate`**, a separate function that wires up scroll listeners, `ResizeObserver`, and (optionally) an animation-frame loop, then calls your update callback.

**Middleware** is the extensibility model. Each middleware is an object with a `fn` that receives the current coordinates and can mutate them or return data. The built-ins are the vocabulary of floating-element behavior:

- `offset` — distance from the reference.
- `flip` — flip to the opposite placement when the preferred side would overflow.
- `shift` — slide along the axis to stay in view (the "clamp to viewport" behavior).
- `size` — resize the floating element to available space (for scrollable menus).
- `arrow` — compute the position of a pointer/caret element.
- `autoPlacement` — choose the placement with the most space (mutually exclusive in spirit with `flip`).
- `hide` — expose whether the reference has scrolled out of view so you can hide the floater.
- `inline` — handle references that span multiple lines (e.g. a wrapped hyperlink).

**Order matters and is a real source of bugs.** Middleware runs left to right; `offset` must come first, and `flip`/`shift` interact — putting `shift` before `flip` produces different (usually wrong) results. The docs prescribe the canonical order and it should be treated as load-bearing.

On the React side, `@floating-ui/react` (formerly `@floating-ui/react-dom-interactions`) adds the behavioral layer: `useHover`, `useClick`, `useFocus`, `useDismiss`, `useRole`, `useListNavigation`, `useTypeahead`, plus `FloatingPortal`, `FloatingFocusManager`, and `FloatingOverlay`. These are what make it possible to build a keyboard-accessible menu or combobox, and they are the reason many component libraries depend on it rather than reimplementing.

## Production Notes

**It is a primitive — you assemble accessibility yourself.** The vanilla and Vue packages give you coordinates and nothing else: no focus trapping, no ARIA, no dismiss-on-escape, no click-outside. A "tooltip" built naively with only `@floating-ui/dom` is likely inaccessible. Budget for the interaction layer, or use `@floating-ui/react` where those hooks exist.

**`autoUpdate` is not free.** With `animationFrame: true` it runs a rAF loop and can dominate a busy page's frame budget if you have many open floaters. Leave it off unless the reference moves continuously (e.g. following an animated element); the default scroll/resize listeners are cheaper. Always call the cleanup function `autoUpdate` returns — forgetting it leaks listeners across mount/unmount cycles.

**Transforms and `offsetParent` surprises.** Positioning reads the offset-parent chain. A CSS `transform`, `filter`, `will-change`, or `contain` on an ancestor creates a new containing block, which shifts what "position: absolute" is relative to. Floating elements inside transformed containers can jump. The common fix is `FloatingPortal` (React) or manually portaling the floater to `document.body` with `strategy: 'fixed'`.

**`position: fixed` vs `absolute`.** The `strategy` option toggles this. `fixed` escapes `overflow: hidden`/`clip` ancestors and is the pragmatic default for portaled content; `absolute` keeps the floater in document flow. Choosing the wrong one is a frequent cause of "my dropdown is clipped inside a scroll container".

**Bundle size / tree-shaking.** The engine is small, but middleware is opt-in — you only pay for what you import. This is deliberate and is a genuine advantage over the older monolithic Popper. Import individual middleware, not a barrel.

**Popper migration.** Moving from Popper 2 is not a drop-in rename. The modifier system became middleware, the API surface changed (`createPopper` → `computePosition` + `autoUpdate`), and update semantics differ (Popper auto-updated by default; Floating UI does not). Follow the official migration guide rather than sed-replacing imports[^2].

**CSS Anchor Positioning is coming for this space.** The native CSS `anchor()` / `position-anchor` feature does viewport-aware anchoring in the browser without JavaScript. As of 2026 it ships in Chromium but not yet across all engines; Floating UI remains the cross-browser answer, but new projects should be aware the platform is absorbing the core use case[^4].

## When to Use / When Not

**Use when:**
- You are building a design system or component library and need positioning you fully control.
- You need collision-aware placement that works across all current browsers today.
- You're on React and want accessible interaction hooks (dismiss, focus management, list navigation) without hand-rolling them.
- You target a non-DOM platform (React Native, Canvas) and can implement a `Platform`.

**Avoid when:**
- You want a ready-made, styled Tooltip/Popover component — reach for a component library that uses Floating UI internally instead.
- You only ship to Chromium and can adopt native CSS Anchor Positioning.
- Your need is a single static tooltip with no collision concerns — plain CSS may be enough.

## Alternatives

- radix-ui/primitives — unstyled, accessible component primitives (Popover, Tooltip, DropdownMenu) that use Floating UI under the hood; use when you want finished behavior, not raw coordinates.
- tailwindlabs/headlessui — headless components with built-in anchoring; use when you're in the Tailwind/React or Vue ecosystem and want fewer moving parts.
- atomiks/tippyjs — higher-level tooltip/popover library built on the Popper lineage; use when you want a configurable tooltip out of the box rather than a primitive.
- floating-ui/floating-ui `v2.x` (Popper) — the legacy branch; use only to maintain existing Popper 2 code, not for new work.
- Native CSS Anchor Positioning — no dependency; use when you can target Chromium-only and want zero JS.

## History

| Version | Date | Notes |
|---------|------|-------|
| Popper 1.0 | 2016 | Original Popper.js; positioning via modifiers[^1]. |
| Popper 2.0 | 2020 | TypeScript rewrite; kept on the `v2.x` branch. |
| Floating UI | 2021–2022 | Rename + re-architecture: `computePosition` core, middleware model, platform abstraction[^2]. |
| `@floating-ui/react` | 2022–2023 | Interaction/accessibility hooks (formerly `@floating-ui/react-dom-interactions`). |

## References

[^1]: Floating UI documentation — "Getting Started" and rationale. https://floating-ui.com/docs/getting-started
[^2]: Floating UI — "Migrating from Popper". https://floating-ui.com/docs/migration
[^3]: Floating UI — "Platform" (custom platform interface for non-DOM targets). https://floating-ui.com/docs/platform
[^4]: MDN — CSS anchor positioning. https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_anchor_positioning

## Tags

typescript, javascript, positioning, tooltip, popover, dropdown, react, ui-primitives, accessibility, frontend, popper
