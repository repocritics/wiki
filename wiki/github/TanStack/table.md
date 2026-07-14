# TanStack/table

> Headless UI for building datagrids: it owns table state and logic, you own every pixel of markup.

[GitHub repo](https://github.com/TanStack/table) ·
[Official website](https://tanstack.com/table) ·
[License: MIT](https://github.com/TanStack/table/blob/main/LICENSE)

## Overview

TanStack Table is a headless table library. It computes and manages the state behind a datagrid — sorting, filtering, grouping, aggregation, pagination, row selection, column visibility and ordering, expansion — but renders nothing. It returns a table instance of rows, cells, and header objects, and you write the `<table>`/`<div>` markup and CSS yourself. This is the defining tradeoff: near-total control over DOM, styling, and accessibility, in exchange for writing (and owning) all of the rendering and layout code that a batteries-included grid would give you for free[^1].

The project began as **react-table** by Tanner Linsley (first commit 2016)[^2]. Through v7 it was a React-only, hooks-based library. Version 8 (2022) was a ground-up TypeScript rewrite that split a framework-agnostic core (`@tanstack/table-core`) from thin per-framework adapters, and the project was renamed **TanStack Table** to reflect that React was now one binding among several[^3]. As of 2026 it is one of the most-used table libraries in the JS/TS ecosystem, with ~28k stars and adapters for React, Vue, Solid, and Svelte; a v9 line (default branch `beta`) is in development and adds further adapters (Angular, Lit, Qwik, Alpine, Ember).

The audience is teams that already have a design system or specific accessibility/markup requirements and do not want a grid's opinions imposed on their DOM. If you want a grid that looks good out of the box, this is the wrong tool — see Alternatives.

## Getting Started

```bash
npm install @tanstack/react-table   # or vue-table / solid-table / svelte-table
```

```tsx
import {
  createColumnHelper,
  flexRender,
  getCoreRowModel,
  useReactTable,
} from "@tanstack/react-table";

type Person = { name: string; age: number };
const columnHelper = createColumnHelper<Person>();

const columns = [
  columnHelper.accessor("name", { header: "Name" }),
  columnHelper.accessor("age", { header: "Age" }),
];

function Table({ data }: { data: Person[] }) {
  const table = useReactTable({ data, columns, getCoreRowModel: getCoreRowModel() });

  return (
    <table>
      <thead>
        {table.getHeaderGroups().map((hg) => (
          <tr key={hg.id}>
            {hg.headers.map((h) => (
              <th key={h.id}>{flexRender(h.column.columnDef.header, h.getContext())}</th>
            ))}
          </tr>
        ))}
      </thead>
      <tbody>
        {table.getRowModel().rows.map((row) => (
          <tr key={row.id}>
            {row.getVisibleCells().map((cell) => (
              <td key={cell.id}>{flexRender(cell.column.columnDef.cell, cell.getContext())}</td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

## Architecture / How It Works

The engine is `@tanstack/table-core`: framework-neutral, no rendering, no reactivity. It takes `data` and `columns`, plus a set of options, and produces a table instance whose methods (`getRowModel`, `getHeaderGroups`, `getState`, `setPageIndex`, …) are pure functions over the current state. The per-framework packages (`@tanstack/react-table`, `-vue-table`, `-solid-table`, `-svelte-table`) are thin adapters: they wire the core's state into that framework's reactivity (React hooks, Vue refs, Solid signals) and re-run the instance when state changes[^3].

**Features are opt-in row models.** Most functionality is delivered as tree-shakeable functions you import and pass explicitly:

- `getCoreRowModel()` — required; the base row list.
- `getSortedRowModel()`, `getFilteredRowModel()`, `getPaginationRowModel()`, `getGroupedRowModel()`, `getExpandedRowModel()`, `getFacetedRowModel()` — each enables the corresponding feature and adds a transformation stage.

Row models chain: filtering feeds grouping feeds sorting feeds pagination, producing the final visible rows. If you don't import a model, that feature's code isn't in your bundle.

**Columns** are defined by an array of column defs. `accessorKey`/`accessorFn` extract cell values; `columnHelper` exists purely to recover TypeScript inference across the generic `TData`/`TValue` types, which are otherwise hard to thread. `display` and `group` columns cover action buttons and multi-row headers.

**State is controlled or uncontrolled.** By default the instance holds its own state internally. Pass `state` + `onStateChange` (or per-slice pairs like `sorting`/`onSortingChange`) to hoist any slice into your own store — the standard pattern for syncing to URL params or server queries.

**`flexRender`** is the seam between headless logic and framework rendering: it invokes a column's `header`/`cell` renderer with the correct context, allowing those renderers to be strings, components, or functions.

There is deliberately **no virtualization** in the core. For large datasets you pair it with `@tanstack/virtual`, a separate library, and render only the visible row window yourself.

## Production Notes

**Referential stability is the number-one footgun.** `data` and `columns` must be stable references across renders. Defining either inline inside a component body (`const columns = [...]` on every render) causes the table to see "new" inputs every render, defeating memoization and, in React, sometimes triggering render loops. The fix is `useMemo`/module-scope constants for `columns` and stable state (from `useState`/store) for `data`. This trips up nearly everyone once.

**Headless means you build a lot.** Sticky headers, resizable columns' actual visual behavior, keyboard navigation, ARIA roles, focus management, virtualization, and all styling are your responsibility. The library gives you the state (`column.getSize()`, `header.getResizeHandler()`), not the DOM. Budget real time for the rendering layer; the "hello world" is short but a production grid is not.

**Server-side data.** For datasets you paginate/sort/filter on the server, set `manualPagination`, `manualSorting`, `manualFiltering` to `true` and provide `rowCount`/`pageCount`. The table then stops doing client-side transforms and just reflects the state you feed it — you own the fetch. Forgetting a `manual*` flag means the table also re-sorts/paginates the already-server-processed page, producing subtly wrong results.

**TypeScript cost.** Inference is a headline feature but the generics are heavy; large column defs with deep row types can slow the type-checker and produce long, hard-to-read error messages. `columnHelper` mitigates but does not eliminate this. Module augmentation is required to add typed `meta` to columns/tables.

**v7 → v8 was a full rewrite**, not an upgrade — the API (hook plugins → row models, different option shapes) is entirely different, and migration is a rewrite of table code, though rendering markup often survives. Teams on v7 (`react-table@7`) should treat a move to v8 as a project, not a bump. The v9 line changes adapter APIs again; check the migration guide before adopting the `beta` branch in production.

**Bundle size** is modest because features are tree-shaken, but the real weight of a shipped grid is the rendering, virtualization, and styling code you add on top.

## When to Use / When Not

**Use when:**
- You have a design system or strict markup/accessibility requirements and need the grid to conform to your DOM, not the other way around.
- You want full control over rendering while offloading sort/filter/group/paginate/select state logic.
- You're multi-framework or want the same mental model across React/Vue/Solid/Svelte.
- You'll pair it with your own (or TanStack) virtualization for large data.

**Avoid when:**
- You want a grid that looks and works well out of the box with minimal code — reach for a batteries-included grid instead.
- You need built-in cell editing, Excel-like features, or canvas rendering for millions of rows.
- The team lacks the time/skill to build and maintain a rendering layer, sticky headers, and accessibility by hand.
- You need enterprise features (range selection, pivoting, integrated charting) that come standard elsewhere.

## Alternatives

- ag-grid/ag-grid — full-featured, styled enterprise datagrid; use when you want editing, pivoting, and range selection out of the box (Enterprise tier is paid).
- mui/mui-x — MUI X DataGrid; use when you're already on Material UI and want a styled, integrated grid (advanced features are paid).
- glideapps/glide-data-grid — canvas-rendered; use when you must display millions of cells with spreadsheet-like performance.
- adazzle/react-data-grid — use when you want a spreadsheet-style, editable React grid with built-in UI.
- silevis/reactgrid — use when you need spreadsheet-like cells and editing rather than a headless engine.

## History

| Version | Date | Notes |
|---------|------|-------|
| react-table v6 | 2018 | HOC/component-based React grid. |
| react-table v7 | 2020 | Hooks + plugin model; introduced the headless approach, React-only[^2]. |
| v8.0 | 2022 | TypeScript rewrite; framework-agnostic core + adapters; renamed TanStack Table[^3]. |
| v9 (beta) | in progress | New adapter APIs; adds Angular/Lit/Qwik/Alpine/Ember; TanStack Intent agent skills[^4]. |

## References

[^1]: TanStack Table docs, "Introduction" — headless UI rationale. https://tanstack.com/table/latest/docs/introduction
[^2]: TanStack Table repository history (originally `react-table`), Tanner Linsley. https://github.com/TanStack/table
[^3]: TanStack Table docs, "Migrating to V8" and core/adapter architecture. https://tanstack.com/table/latest/docs/guide/migrating
[^4]: Repository README — v9 adapters and `@tanstack/intent` agent skills. https://github.com/TanStack/table
[^5]: TanStack Virtual — separate virtualization library paired with Table. https://tanstack.com/virtual

## Tags

typescript, javascript, datagrid, table, headless-ui, react, vue, solid, svelte, frontend, ui-library
