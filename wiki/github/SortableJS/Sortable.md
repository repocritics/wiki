# SortableJS/Sortable

> Framework-agnostic vanilla-JS library for reorderable drag-and-drop lists, with a native-DnD path and a mouse/touch fallback.

[GitHub repo](https://github.com/SortableJS/Sortable) ·
[Official website](https://sortablejs.github.io/Sortable/) ·
[License: MIT](https://github.com/SortableJS/Sortable/blob/master/LICENSE)

## Overview

Sortable is a dependency-free JavaScript library for building reorderable lists
via drag-and-drop, including dragging items between lists. First published in
December 2013[^1], it predates most current framework-specific DnD libraries and
became the default vanilla answer to "make this list sortable." With over 31,000
stars and ~3,700 forks it is one of the most-depended-on drag-and-drop libraries
on npm, pulled in transitively by countless admin panels, kanban boards, and
form builders.

Its defining design choice is a dual input model: by default it uses the native
HTML5 Drag and Drop API, but it ships a "fallback" mode that emulates dragging
with pointer/touch events and a cloned ghost element. This exists because native
HTML5 DnD is inconsistent across browsers and effectively unusable on touch
devices — so in practice many production users force the fallback path on
(`forceFallback: true`) to get predictable behavior. Understanding which path you
are on explains most of the library's quirks.

The other defining tension is its relationship to component frameworks. Sortable
mutates the real DOM directly — it physically moves nodes — which collides with
the virtual DOM in React and Vue. The core library is framework-agnostic by
design, and integration is pushed out to separate wrapper repositories of
varying maintenance health. As of mid-2026 the core repo's last commit was in
March 2026 with ~520 open issues, reflecting thin maintenance relative to its
enormous install base.

## Getting Started

```bash
npm install sortablejs
# TypeScript types are separate: npm install -D @types/sortablejs
```

```html
<ul id="items">
  <li>item 1</li>
  <li>item 2</li>
  <li>item 3</li>
</ul>
```

```js
import Sortable from 'sortablejs';

const el = document.getElementById('items');
Sortable.create(el, {
  animation: 150,        // ms; 0 disables move animations
  handle: '.drag-handle',// optional: restrict drag to a sub-element
  onEnd(evt) {
    // evt.oldIndex / evt.newIndex give the reordering
    console.log(evt.oldIndex, '->', evt.newIndex);
  },
});
```

Any container/child elements work, not just `ul`/`li`. A UMD build is also
available for `<script>` tags via CDN (jsDelivr/unpkg) exposing a global
`Sortable`.

## Architecture / How It Works

The core is a single class (`src/Sortable.js`) with no runtime dependencies. On
`Sortable.create(el, options)` it wires listeners on the container and, when a
drag starts, tracks a `dragEl`, computes swap zones on each `dragover`/pointer
move, and reinserts the node before or after the target based on the pointer
position within a configurable `swapThreshold`. It emits a lifecycle of
callbacks (`onChoose`, `onStart`, `onMove`, `onChange`, `onEnd`, plus cross-list
`onAdd`/`onRemove`/`onUpdate`/`onSort`) that report `oldIndex`/`newIndex` and the
source/target lists.

Two input paths coexist:

- **Native path** — uses the HTML5 `dragstart`/`dragover`/`drop` events. Cannot
  be delayed on some browsers, cannot style the drag image consistently, and
  does not work on touch.
- **Fallback path** — listens to `mousedown`/`touchstart`, clones the dragged
  element into a positioned "fallback" node, and moves it manually. Enabled
  automatically when native DnD is unavailable, or forced with `forceFallback`.

Since the 1.8 line, optional behavior is factored into **plugins** mounted with
`Sortable.mount()`: `AutoScroll` (scroll the container/window near edges),
`MultiDrag` (select and drag several items at once), `Swap` (swap two items
instead of shifting), and `OnSpill`. The default bundle includes AutoScroll;
`sortable.core.esm.js` ships without default plugins, and
`sortable.complete.esm.js` bundles everything, so bundle size is a function of
which entry point you import.

Cross-list dragging is governed by the `group` option (`name` + `pull`/`put`
rules), which also supports `clone` mode where the source keeps a copy. The
`store` interface (get/set) is the built-in hook for persisting order to
`localStorage` or a backend via `toArray()`.

## Production Notes

- **Prefer the fallback for touch and custom visuals.** Native HTML5 DnD does
  not fire on touch devices and gives you almost no control over the drag image.
  Many teams set `forceFallback: true` globally for consistent desktop/mobile
  behavior; treat the native path as the exception, not the default.
- **Framework wrappers are separate repos with independent (and uneven) health.**
  `react-sortablejs`, `ngx-sortablejs`, and the Vue integration all live outside
  this repo. The Vue story is a known footgun: the original `Vue.Draggable` /
  `vuedraggable` targets Vue 2, and Vue 3 users must find the maintained
  successor rather than the top search result. Check the wrapper's last-release
  date before adopting.
- **Sortable fights the virtual DOM.** Because it moves real DOM nodes, using the
  core directly inside React/Vue without the wrapper (or without resetting the
  DOM and driving order from state) produces duplicated or "ghost" rows after a
  drop. The wrappers exist specifically to reconcile the imperative mutation with
  the declarative render; don't hand-roll it unless you understand the reset
  dance.
- **No keyboard accessibility.** Reordering is pointer/touch only. There is no
  built-in keyboard interaction or ARIA live-region support, which is a real
  a11y gap for any list that must be operable without a mouse.
- **Large lists get expensive.** Each move recomputes rectangles and runs the
  FLIP-style animation; `animation: 0` and avoiding deeply nested sortables
  helps on long lists.
- **Maintenance is thin relative to usage.** The core repo carries a large open
  issue backlog and infrequent releases; the 1.15.x line has been the de facto
  stable version for an extended period. It is mature and widely battle-tested,
  but do not expect rapid fixes for edge cases.

## When to Use / When Not

**Use when:**
- You need drag-to-reorder in plain JS or any framework and want one small,
  dependency-free library that handles native DnD, touch, and cross-list moves.
- You want proven cross-list group/clone semantics and a simple persistence hook.
- You're incrementally enhancing server-rendered HTML (Rails, Django, Laravel,
  htmx) where a vanilla library beats a framework-bound one.

**Avoid when:**
- Keyboard accessibility is a requirement — Sortable has none built in.
- You're all-in on React and want an actively maintained, a11y-aware toolkit —
  reach for dnd-kit instead.
- You need tight integration with a reactive framework's state model and don't
  want to manage the imperative-DOM/virtual-DOM reconciliation yourself.

## Alternatives

- clauderic/dnd-kit — modern, actively maintained React DnD toolkit with keyboard
  accessibility; use instead when you're React-only and a11y matters.
- atlassian/react-beautiful-dnd — opinionated accessible list DnD for React, but
  in maintenance mode/deprecated; use only for legacy React lists, prefer dnd-kit
  for new work.
- react-dnd/react-dnd — lower-level React DnD with pluggable backends; use when
  you need custom drag semantics beyond simple list reordering.
- Shopify/draggable — comparable framework-agnostic vanilla library, more modular
  API; use when you want a similar footprint with a different API surface.
- formkit/drag-and-drop — newer small, framework-agnostic reactive DnD library;
  use when you want data-first reordering with first-class React/Vue bindings.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-12 | First release; HTML5 Drag'n'Drop based reorderable lists[^1]. |
| 1.0 | 2014-12 | First stable line; cross-list groups, handles, events[^2]. |
| 1.8 | 2019 | Plugin architecture (`Sortable.mount`); MultiDrag, Swap, AutoScroll extracted as plugins. |
| 1.10 | 2019 | Consolidated modular/ESM builds and plugin entry points. |
| 1.15 | 2021 | Current stable line; long-lived with infrequent point releases. |

## References

[^1]: "Sorting with the help of HTML5 Drag'n'Drop API" — SortableJS wiki, 2013-12-23. https://github.com/SortableJS/Sortable/wiki/Sorting-with-the-help-of-HTML5-Drag'n'Drop-API/
[^2]: "Sortable v1.0 — New capabilities" — SortableJS wiki, 2014-12-22. https://github.com/SortableJS/Sortable/wiki/Sortable-v1.0-%E2%80%94-New-capabilities/
[^3]: README and options reference. https://github.com/SortableJS/Sortable/blob/master/README.md

## Tags

javascript, drag-and-drop, sortable, reordering, ui-library, touch, vanilla-js, frontend, html5-dnd, framework-agnostic
