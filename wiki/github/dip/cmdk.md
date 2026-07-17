# dip/cmdk

> Fast, unstyled command-menu React component that filters and sorts items for you — the engine behind most "⌘K" palettes and shadcn/ui's Command.

[GitHub repo](https://github.com/pacocoursey/cmdk) ·
[npm](https://www.npmjs.com/package/cmdk) ·
[License: MIT](https://github.com/pacocoursey/cmdk/blob/main/LICENSE.md)

## Overview

cmdk is a single-purpose React component: a composable command menu that doubles as an accessible combobox[^1]. You render `Command.Item` children (statically, or mapped from data, or wrapped in your own components) and the library filters, scores, and sorts them as the user types. It ships no styles — every part exposes a `cmdk-*` data attribute you hook CSS onto — which is why it disappears into so many different-looking product UIs.

It was written by Paco Coursey, originally as a 2019 experiment to see whether a fully composable combobox API was feasible, then rewritten from scratch in 2022 with a simpler, more performant core[^2]. Its reach is outsized relative to its ~12.8k stars because it is the underlying engine of shadcn/ui's `Command` component, so a large fraction of React apps ship cmdk transitively without depending on it directly. The GitHub repo now resolves under the owner `dip` (a maintainer/handle change); the npm package name remains `cmdk` and imports are unchanged.

The defining tradeoff is scope discipline: cmdk does one thing (filter/sort/select a flat-or-grouped list with keyboard nav) and deliberately refuses adjacent features — no virtualization, no built-in styles, no React Native, no automatic ⌘K key listener. That keeps it small and unopinionated, but means anything beyond a few thousand items or any non-web target is your problem to solve.

## Getting Started

```bash
pnpm install cmdk   # or npm / yarn install cmdk
```

```tsx
import { Command } from 'cmdk'

export function CommandMenu() {
  return (
    <Command label="Command Menu">
      <Command.Input placeholder="Type a command…" />
      <Command.List>
        <Command.Empty>No results found.</Command.Empty>
        <Command.Group heading="Letters">
          <Command.Item onSelect={(v) => console.log('selected', v)}>a</Command.Item>
          <Command.Item>b</Command.Item>
          <Command.Separator />
          <Command.Item keywords={['fruit']}>Apple</Command.Item>
        </Command.Group>
      </Command.List>
    </Command>
  )
}
```

For the classic modal palette, wrap in `Command.Dialog` and bind the key yourself — cmdk intentionally does not register `⌘K` for you, so you control the keybind context:

```tsx
const [open, setOpen] = React.useState(false)
React.useEffect(() => {
  const down = (e: KeyboardEvent) => {
    if (e.key === 'k' && (e.metaKey || e.ctrlKey)) { e.preventDefault(); setOpen(o => !o) }
  }
  document.addEventListener('keydown', down)
  return () => document.removeEventListener('keydown', down)
}, [])
```

## Architecture / How It Works

State lives in a single external store at the `Command` root, read through React 18's `useSyncExternalStore`[^3]. `Command.Item`, `Command.Group`, and `Command.Input` register themselves with that store via context on mount, so the tree can be arbitrarily composed — items can be wrapped in your own components, conditionally rendered, or emitted as static JSX and still participate in filtering.

Each item carries a `value`: either the explicit `value` prop, or one inferred from its rendered `.textContent`. Selection is tracked by that value string, not by DOM index — which is what lets the active item stay stable while the list reorders under it. On every search change, cmdk scores each item with a filter function (the default is a fuzzy substring scorer over the value plus any `keywords` aliases), then **imperatively reorders the actual DOM nodes** to reflect ranking rather than re-rendering the list in sorted order. Non-matching items are hidden; groups are never unmounted, they just receive the `hidden` attribute. All values are `trim()`-ed before comparison.

This imperative DOM manipulation is the crux of both its speed and its caveats: it sidesteps React's reconciliation for the sort, which keeps typing responsive, but it is exactly the kind of side-channel the maintainers flag as only "maybe" concurrent-mode safe. `Command.Dialog` composes Radix UI's Dialog primitive for the modal/overlay/portal/focus-trap behavior, so the dialog variant inherits Radix's accessibility and its dependency weight. The escape hatch for advanced cases is `useCommandState`, a hook over `useSyncExternalStore` that subscribes to a slice of internal state (e.g. current `search`) for custom empty states or search-conditional sub-items.

## Production Notes

- **No virtualization — hard ceiling around 2,000–3,000 items.** Beyond that, filtering/sorting and the DOM reorder get visibly slow[^1]. The supported path is `shouldFilter={false}`: you filter and slice the list yourself, pass only what should render, and bring your own virtualizer (e.g. TanStack Virtual). This is also the more memory-efficient mode for large datasets.
- **Unstyled means unstyled.** Out of the box it renders unstyled DOM; you must supply CSS against the `[cmdk-*]` selectors. There are drop-in starter stylesheets in the repo, but there is no theme. Teams usually adopt shadcn/ui's pre-styled `Command` wrapper instead of styling raw cmdk.
- **Every item needs a unique, stable `value` and a `key`.** Duplicate or missing values are the single most common source of "weird/wrong behavior" reports — two items with the same inferred text will collide on selection. Remember values are trimmed, so leading/trailing whitespace won't disambiguate.
- **SSR/hydration:** pass `open={false}` to `Command.Dialog` on the server; a truthy initial `open` is the usual cause of hydration mismatch. React 18 is required (it relies on `useId` and `useSyncExternalStore`) — it will not work on React 17.
- **Async items** render as they arrive and are filtered automatically; gate a `Command.Loading` on your loading flag. There is no built-in debounce or request cancellation — that's caller-side.
- **Bundle:** the core is small, but `Command.Dialog` pulls in Radix Dialog. If you render cmdk inside a popover, use `@radix-ui/react-popover` so the Radix dependency is shared rather than duplicated.
- **Maintenance cadence has slowed.** The last repository push was late October 2025; the API has been effectively stable since 1.0 and the project reads as "done/quiet" rather than actively iterating. For a small, single-purpose component this is largely fine, but expect slow turnaround on issues and don't bank on new features.

## When to Use / When Not

**Use when:**
- You want a keyboard-navigable command palette or combobox and full control over its appearance.
- Your list is up to a few thousand items, or you're willing to run `shouldFilter={false}` with your own virtualization.
- You're already in the shadcn/ui or Radix ecosystem (cmdk is the native fit).
- You need an accessible combobox/dialog without hand-rolling ARIA and focus management.

**Avoid when:**
- You have tens of thousands of items and don't want to build the filtering/virtualization layer yourself.
- You need React Native or a non-DOM renderer — unsupported, with no plans.
- You want batteries-included actions, registries, and theming out of the box (kbar is closer to that).
- You're on React 17 or need guaranteed concurrent-mode/`<StrictMode>`-hardened internals — the imperative DOM ordering is an acknowledged risk area.

## Alternatives

- timc1/kbar — batteries-included command-bar with an action registry, built-in animations and theming; choose it when you want an opinionated ⌘K experience rather than a bare primitive.
- downshift-js/downshift — headless autocomplete/combobox/select primitives; choose it when you need a standards-strict combobox/select and not a command-menu shape.
- ariakit/ariakit — broad accessible component kit with Combobox and Menu building blocks; choose it when you want one library covering many widgets, not a single component.
- radix-ui/primitives — unstyled Dialog/Popover/etc.; choose it when you'd rather assemble your own menu from lower-level accessible parts.
- tailwindlabs/headlessui — accessible Combobox/Dialog for React and Vue; choose it when you're standardizing on Headless UI and don't need palette-style scoring.

## History

| Version | Date | Notes |
|---------|------|-------|
| prototype | 2019 | First experiment by Paco Coursey; `use-descendants` later extracted from it[^2]. |
| — | 2020 | Used for Vercel's command menu and autocomplete[^2]. |
| 0.1.x | 2022-07 | Independent rewrite published to npm; simpler, more performant core[^2]. |
| 1.0.0 | 2024 | Stable API; React 18 required (`useId`, `useSyncExternalStore`). |
| (quiet) | 2025-10 | Last repository push; API stable, cadence slowed[^4]. |

## References

[^1]: cmdk README — API, FAQ (accessibility, virtualization ceiling, React 18 requirement). https://github.com/pacocoursey/cmdk#readme
[^2]: cmdk README, "History" — 2019 prototype, 2020 Vercel usage, 2022 rewrite, `use-descendants` extraction. https://github.com/pacocoursey/cmdk#history
[^3]: cmdk `useCommandState` / composition notes (composes `useSyncExternalStore`). https://github.com/pacocoursey/cmdk#usecommandstate
[^4]: GitHub repository metadata (owner now `dip`, `pushed_at` 2025-10-29, MIT, ~12.8k stars) via `gh api repos/pacocoursey/cmdk`, fetched 2026-07.

## Tags

react, typescript, command-palette, combobox, command-menu, ui-component, headless-ui, accessibility, radix-ui, frontend
