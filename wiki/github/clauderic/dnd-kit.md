# clauderic/dnd-kit

> A drag-and-drop toolkit for the web, built around accessibility and a
> sensor/collision architecture rather than the native HTML5 DnD API.

[GitHub repo](https://github.com/clauderic/dnd-kit) ·
[Official website](https://dndkit.com) ·
[License: MIT](https://github.com/clauderic/dnd-kit/blob/main/LICENSE)

## Overview

dnd-kit is a drag-and-drop library authored by Claudéric Demers, first
published to npm in January 2021[^1]. It exists because the two obvious
alternatives at the time were both compromised: the native HTML5 Drag and Drop
API is famously inconsistent and nearly impossible to make accessible, and
`react-beautiful-dnd` — the other popular React option — was list-only and
heading toward maintenance mode. dnd-kit's pitch was a small, dependency-free
core that does not use the HTML5 DnD API at all, layered under React hooks
(`useDraggable`, `useDroppable`, `useSortable`) with keyboard and screen-reader
support treated as first-class rather than bolted on. With ~17k stars it is now
one of the most-used drag-and-drop libraries in the React ecosystem.

The defining tension in 2026 is a **version split**. The version almost
everyone runs in production is v6 — the React-only packages `@dnd-kit/core`,
`@dnd-kit/sortable`, `@dnd-kit/modifiers`, and `@dnd-kit/utilities`. Its last
stable release, 6.3.1, shipped in December 2024[^2] and has been quiet since.
Meanwhile the repository's `main` branch is a ground-up rewrite: a
framework-agnostic core (`@dnd-kit/abstract`) with a DOM layer (`@dnd-kit/dom`)
and thin adapters for React, Vue, Svelte, and Solid[^3]. That rewrite is still
published under 0.x version numbers (`@dnd-kit/react` was 0.5.0 as of mid-2026)
and is not API-compatible with v6. So the docs on GitHub describe one thing;
the package most teams `npm install` is the other.

## Getting Started

The stable, production path is the React v6 packages:

```bash
npm install @dnd-kit/core @dnd-kit/sortable @dnd-kit/modifiers @dnd-kit/utilities
```

```tsx
// A minimal sortable list (v6 / @dnd-kit React API)
import { DndContext, closestCenter, PointerSensor, useSensor, useSensors } from "@dnd-kit/core";
import { arrayMove, SortableContext, useSortable, verticalListSortingStrategy } from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";

function Item({ id }: { id: string }) {
  const { attributes, listeners, setNodeRef, transform, transition } = useSortable({ id });
  const style = { transform: CSS.Transform.toString(transform), transition };
  return <li ref={setNodeRef} style={style} {...attributes} {...listeners}>{id}</li>;
}

export function List() {
  const [items, setItems] = useState(["a", "b", "c"]);
  const sensors = useSensors(useSensor(PointerSensor));
  return (
    <DndContext sensors={sensors} collisionDetection={closestCenter}
      onDragEnd={({ active, over }) => {
        if (over && active.id !== over.id)
          setItems((it) => arrayMove(it, it.indexOf(active.id as string), it.indexOf(over.id as string)));
      }}>
      <SortableContext items={items} strategy={verticalListSortingStrategy}>
        <ul>{items.map((id) => <Item key={id} id={id} />)}</ul>
      </SortableContext>
    </DndContext>
  );
}
```

## Architecture / How It Works

dnd-kit does not use the browser's native drag events. Instead it is built from
composable pieces you assemble yourself:

- **Sensors** translate raw input into drag lifecycle events. `PointerSensor`,
  `MouseSensor`, `TouchSensor`, and `KeyboardSensor` ship built-in; each takes
  *activation constraints* (a movement distance or a press delay) that decide
  when a gesture counts as a drag versus a click or scroll.
- **`DndContext`** is the coordinator. It owns the collision graph, tracks the
  active draggable, and fires `onDragStart` / `onDragMove` / `onDragOver` /
  `onDragEnd`. Nothing moves on screen unless you make it move — the library
  reports intent; you apply the `transform`.
- **Collision detection** is a pluggable function (`closestCenter`,
  `closestCorners`, `rectIntersection`, `pointerWithin`). The choice is not
  cosmetic: nested or overlapping droppables behave very differently under each,
  and picking the wrong one is a common source of "it drops in the wrong place"
  bugs.
- **`DragOverlay`** renders the dragged item in a portal detached from document
  flow, which avoids the layout-shift and `overflow: hidden` clipping problems
  you hit when transforming the original node in place.
- **`@dnd-kit/sortable`** is a thin layer on top: `SortableContext` plus a
  *strategy* (vertical list, horizontal list, rect grid) that computes where
  siblings should animate to.

The v6 core is dependency-free and tree-shakeable. The rewrite on `main`
formalizes the layering into separate packages — `@dnd-kit/abstract`,
`@dnd-kit/dom`, `@dnd-kit/collision`, `@dnd-kit/geometry`, `@dnd-kit/state` —
so the same engine can drive Vue, Svelte, and Solid, not just React[^3]. The
monorepo builds with Turborepo and bun.

## Production Notes

**The stable version is effectively frozen while the rewrite lands.**
`@dnd-kit/core` 6.3.1 (Dec 2024) is the newest stable release; the
framework-agnostic packages are still 0.x. New projects must choose between a
mature-but-stagnant v6 and an actively-developed-but-pre-1.0 rewrite whose API
will keep shifting. Treat the 0.x packages as beta.

**Virtualization is not free.** dnd-kit measures droppable rectangles; combining
it with a windowing library (`@tanstack/react-virtual`, `react-window`) means
off-screen items are unmounted and therefore unmeasured. Large sortable lists
need deliberate work — custom collision detection, or keeping measured — and
this is one of the most-reported friction points.

**Touch devices need CSS help.** Because dnd-kit intercepts pointer/touch
events, dragging fights native scroll. The standard fix is `touch-action: none`
on drag handles plus a `TouchSensor` activation delay, so tap-and-hold drags
while a swipe still scrolls. Getting this wrong makes mobile lists feel broken.

**Immutable reordering is on you.** `useSortable` needs stable, unique `id`s and
expects you to reorder your own state (typically via `arrayMove`) in
`onDragEnd`. Index-based keys or in-place mutation produce dropped animations
and mis-sorts. Likewise, prefer `DragOverlay` for anything non-trivial —
transforming the source node in place interacts badly with `position`,
`overflow`, tables, and grids.

**Migration is a rewrite, not an upgrade.** Moving from v6 to the new
`@dnd-kit/react` changes hooks, package names, and mental model. Budget it as
porting work, and pin versions until the new line reaches 1.0.

## When to Use / When Not

**Use when:**
- You need accessible drag-and-drop — keyboard operability, ARIA attributes, and
  screen-reader live regions are built in, not an afterthought.
- You want fine control (custom sensors, collision algorithms, modifiers) rather
  than a batteries-included list widget.
- You're on React today and comfortable staying on v6 until the rewrite settles.

**Avoid when:**
- You want a drop-in sortable list with zero assembly — dnd-kit is a toolkit;
  you wire the pieces together yourself.
- You need a stable, actively-released API right now and can't accept either a
  frozen v6 or a 0.x rewrite.
- You're rendering very large virtualized lists and don't want to hand-tune
  collision/measurement.

## Alternatives

- atlassian/pragmatic-drag-and-drop — framework-agnostic, built on the native
  HTML5 DnD API; use when you need proven performance on huge lists (it powers
  Trello/Jira) and don't mind a lower-level API.
- atlassian/react-beautiful-dnd — polished list reordering UX, but deprecated
  and no longer maintained; use only for legacy code, otherwise prefer its
  successor Pragmatic DnD.
- SortableJS/Sortable — mature imperative vanilla-JS library; use when you're not
  in a component framework or want the smallest imperative option.
- react-dnd/react-dnd — older, backend-abstraction model (HTML5/touch); use when
  you need its established ecosystem and don't require built-in accessibility.
- formkit/drag-and-drop — small, newer, data-first API; use when you want minimal
  setup for simple sortable lists.

## History

| Version | Date | Notes |
|---------|------|-------|
| @dnd-kit/core 1.0.0 | 2021-01-02 | First npm publish[^1]. |
| 3.0.0 | 2021-04-20 | Early API churn during 2021. |
| 4.0.0 | 2021-09-28 | Continued core/sortable refinement. |
| 5.0.0 | 2022-01-10 | Pre-6 stabilization. |
| 6.0.0 | 2022-05-21 | The long-lived stable React line[^2]. |
| 6.3.1 | 2024-12-05 | Last stable v6 release to date[^2]. |
| @dnd-kit/react 0.x | 2024–2026 | Framework-agnostic rewrite; adapters for React, Vue, Svelte, Solid; still pre-1.0[^3]. |

## References

[^1]: npm registry, `@dnd-kit/core` — first version 1.0.0 published 2021-01-02.
  https://www.npmjs.com/package/@dnd-kit/core
[^2]: npm registry, `@dnd-kit/core` version history — 6.0.0 (2022-05-21) through
  6.3.1 (2024-12-05, latest stable). https://www.npmjs.com/package/@dnd-kit/core?activeTab=versions
[^3]: dnd-kit README and package listing — `@dnd-kit/abstract`, `@dnd-kit/dom`,
  and framework adapters (React/Vue/Svelte/Solid). https://github.com/clauderic/dnd-kit

## Tags

typescript, react, drag-and-drop, sortable, accessibility, frontend, ui-toolkit, hooks, vue, svelte, solidjs, monorepo
