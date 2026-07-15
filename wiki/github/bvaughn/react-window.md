# bvaughn/react-window

> React components that render only the visible slice of a large list or grid, keeping the DOM small no matter how much data you have.

[GitHub repo](https://github.com/bvaughn/react-window) ·
[Official website](https://react-window.vercel.app/) ·
[License: MIT](https://github.com/bvaughn/react-window/blob/main/LICENSE.md)

## Overview

`react-window` is a windowing (virtualization) library for React by Brian
Vaughn, a former React core team member[^1]. The idea is narrow: rendering
10,000 rows means 10,000 DOM nodes, layout, and paint; windowing renders only
the rows currently in the viewport (plus a small overscan buffer) and
repositions them with absolute positioning as the user scrolls. The visible
node count stays roughly constant, so scroll performance and memory stop
scaling with dataset size.

It is the deliberately smaller successor to Vaughn's earlier
`react-virtualized`. That library accumulated a large surface (tables, masonry,
`WindowScroller`, `AutoSizer`, collections) and a correspondingly large bundle;
`react-window` is a ground-up rewrite that keeps only the core primitives —
lists and grids — and pushes everything else to userland or companion
packages[^2]. The defining tension is exactly that scope decision: you trade
batteries-included features for a small, fast, predictable core. It ships in
production in React DevTools and the Replay debugger, among others.

The 2.x line (the current major, reflected in the docs and README) is itself an
API rewrite of the original 1.x. Where 1.x exposed four components
(`FixedSizeList`, `VariableSizeList`, `FixedSizeGrid`, `VariableSizeGrid`), 2.x
collapses them into a single `List` and single `Grid` that accept a `rowHeight`
/ `columnWidth` number, string percentage, or function[^3]. This is a hard
breaking change; 1.x docs are kept on a separate site.

## Getting Started

```sh
npm install react-window
```

```tsx
import { List, type RowComponentProps } from "react-window";

function Row({ index, style }: RowComponentProps) {
  return <div style={style}>Row {index}</div>;
}

export default function App() {
  return (
    <List
      rowComponent={Row}
      rowCount={100000}
      rowHeight={32}
      rowProps={{}}
      style={{ height: 400 }}
    />
  );
}
```

`rowComponent` receives an `index` and a `style` prop; the `style` (absolute
position + height) **must** be spread onto the row's outer element — this is how
react-window places rows in the scroll container. Anything in `rowProps` is
forwarded to every row and triggers a re-render when it changes.

## Architecture / How It Works

The core is a scroll container with a single tall inner element whose height
equals `rowCount × rowHeight` (or the summed heights for variable sizing). On
scroll, the component computes the first and last visible index from
`scrollTop`, height, and item sizes, then renders only that range. Each rendered
row is absolutely positioned via the injected `style`. For fixed sizes the
start-index math is O(1) division; for variable sizes react-window maintains a
cache of measured/estimated offsets so lookups stay fast as you scroll.

`List` supports dynamic row heights through the `useDynamicRowHeight` hook,
which measures rendered rows and feeds a height cache back to the list. The docs
are explicit that this is slower than known sizes and recommend supplying
heights ahead of time when possible[^3]. `Grid` is stricter: cell sizes must be
known ahead of time (static, or derivable from `cellProps` without rendering) —
there is no dynamic-measurement path for grids.

In 2.x the `List`/`Grid` measure their own container via a resize observer and
expose an `onResize` callback, so the component fills the size you give it
through `style` — folding in a job that in 1.x required the separate
`react-virtualized-auto-sizer` package. Scrolling and programmatic control go
through an imperative handle (`listRef` / `gridRef`) rather than props. What
react-window deliberately omits: sticky headers, a table abstraction,
infinite-loading, sortable columns. Those are left to companion packages
(`react-window-infinite-loader`) or userland composition.

## Production Notes

**Off-screen content does not exist in the DOM.** Because unrendered rows are
not mounted, native Ctrl+F / find-in-page will not match them, screen readers
see only the rendered window, and `Cmd+A` copy grabs only visible rows. For
lists that must be fully searchable or selectable, virtualization is the wrong
tool. Default ARIA roles cover simple cases; complex grids need attention.

**Dynamic heights are the main footgun.** Every measured-height row means a
render-then-measure-then-reposition cycle, which can cause visible layout shift
and scroll jumps when rows above the viewport resize. Provide known heights
whenever the data allows it. Images and rich text that reflow after load are the
usual culprits behind "the scrollbar jumps" bug reports.

**The container needs a bounded height.** A `List` in an unbounded-height parent
renders nothing or one row — give it an explicit pixel height (via `style`, or a
sized parent it can measure). This is the most common first-use mistake. `Grid`
adds the constraint that both axes need sizes up front: there is no per-cell
measurement escape hatch, so truly content-sized cells are out of scope.

**Migrations are real breaks, not upgrades.** 1.x → 2.x is a full API rewrite:
the `FixedSizeList` / `VariableSizeList` / `*Grid` components are gone, replaced
by unified `List` / `Grid` with a different prop shape (`rowComponent`,
`rowProps`, height-as-function). Code, types, and the imperative API all change.
Treat it as adopting a new library, and keep 1.x docs bookmarked for legacy apps.
The project is maintained but low-churn — a stable, sponsor-funded core rather
than a fast-moving framework.

## When to Use / When Not

**Use when:**
- You render long, uniform-ish lists or grids (thousands+ of rows) and DOM node
  count is your bottleneck.
- Item sizes are known ahead of time, or close enough to estimate.
- You want a small dependency with a narrow, well-understood surface.

**Avoid when:**
- Content must be natively searchable/selectable across the whole dataset
  (off-screen rows aren't in the DOM).
- Rows have wildly variable, content-driven heights that can't be measured
  cheaply — auto-measuring virtualizers handle this with less pain.
- Your list is short (a few hundred simple rows); plain rendering is simpler and
  fast enough, and virtualization adds complexity and edge cases.
- You need tables, sticky headers, or infinite loading out of the box.

## Alternatives

- TanStack/virtual — headless, framework-agnostic virtualization primitive; use
  when you want full control over markup/behavior and don't mind wiring it up.
- petyosi/react-virtuoso — auto-measures dynamic content; use when row heights
  are unknown/variable and you want it to "just work" without a measurement hook.
- bvaughn/react-virtualized — the heavier predecessor with tables, masonry, and
  more built-ins; use when you need those features and accept the larger bundle.
- inokawa/virta — small, modern virtualizer with good dynamic-size handling; use
  as a lighter react-virtuoso-style option.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-05 | Repository created; rewrite of react-virtualized's core[^2]. |
| 1.0 | 2019 | First stable release: FixedSize/VariableSize List and Grid[^1]. |
| 2.x | ~2025 | API rewrite to unified `List` / `Grid`, `useDynamicRowHeight`, built-in resize observation (2.x date approximate)[^3]. |

## References

[^1]: react-window documentation and API reference. https://react-window.vercel.app/
[^2]: Brian Vaughn, "react-window" vs "react-virtualized" comparison. https://github.com/bvaughn/react-window#readme
[^3]: react-window List / Grid props and `useDynamicRowHeight` (v2 docs). https://react-window.vercel.app/

## Tags

react, virtualization, windowing, performance, list, grid, frontend, typescript, ui-components, rendering
