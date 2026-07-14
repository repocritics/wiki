# radix-ui/primitives

> Unstyled, accessible React component primitives — you own every pixel, they own the ARIA, focus, and keyboard behavior.

[GitHub repo](https://github.com/radix-ui/primitives) ·
[Official website](https://radix-ui.com/primitives) ·
[License: MIT](https://github.com/radix-ui/primitives/blob/main/LICENSE)

## Overview

Radix Primitives is a library of low-level, unstyled React components — dialogs, dropdown menus, tooltips, popovers, tabs, accordions, sliders, and roughly two dozen more. Each ships the parts that are genuinely hard to get right (WAI-ARIA roles and attributes, focus management, keyboard interaction, collision-aware positioning, controlled/uncontrolled state) and ships zero visual styling. You bring the CSS. It was created by Modulz and is now maintained by WorkOS[^1].

The defining tradeoff is right there in the name: these are *primitives*, not a design system. Unlike Material UI or Ant Design, dropping in a Radix component gives you a correct but visually blank widget. That is the point — Radix is meant to be the accessibility-and-behavior layer under *your* design system, not a look you adopt. The cost is that "install and use" is not the whole story; you write styling for every state, and the compound-component API is more verbose than a single styled `<Modal>`.

Radix's real cultural footprint is indirect: it is the foundation shadcn/ui is built on, which made Radix the de facto behavioral layer for a large slice of the React ecosystem after 2023. Most developers who "use Radix" arrived through copied shadcn components without picking the library directly. Note the naming: this repo (`primitives`, the unstyled library) is distinct from Radix Themes (a styled component library) and Radix Colors (a color-scale system), which live in separate repos under the same org.

## Getting Started

```bash
# unified package (recommended since 2024) — one dependency, namespaced imports
npm install radix-ui
# or the classic per-component packages
npm install @radix-ui/react-dialog
```

```tsx
import { Dialog } from "radix-ui";

export function Example() {
  return (
    <Dialog.Root>
      <Dialog.Trigger>Open</Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Overlay className="overlay" />
        <Dialog.Content className="content">
          <Dialog.Title>Settings</Dialog.Title>
          <Dialog.Description>Adjust your preferences.</Dialog.Description>
          <Dialog.Close>Done</Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

The `className`s do nothing until you write CSS. Radix exposes state as `data-*` attributes (e.g. `[data-state="open"]`) so you style transitions and variants from those hooks.

## Architecture / How It Works

Every component is a **compound component**: a `Root` that holds state via React Context, plus named parts (`Trigger`, `Content`, `Item`, `Portal`, `Close`) that read that context. This is why the API is verbose but composable — you can reorder, wrap, or omit parts and place your own markup between them.

Several cross-cutting internals do the heavy lifting and are reused across components:

- **`asChild` + Slot** — instead of rendering its own DOM node, a part can merge its props, refs, and event handlers onto the single child you provide (`<Dialog.Trigger asChild><MyButton/></Dialog.Trigger>`). This avoids wrapper-element pollution and is the primary composition escape hatch.
- **Controlled/uncontrolled state** — a shared `useControllableState` hook lets every stateful component work either mode from the same `value`/`defaultValue`/`onValueChange` prop triad.
- **Presence** — keeps a component mounted through its exit animation by watching `data-state` and CSS animation/transition end, so you can animate close without a JS animation library.
- **DismissableLayer / FocusScope / RovingFocusGroup** — the shared machinery for outside-click and Escape dismissal, focus trapping, and arrow-key roving through composite widgets.
- **Popper positioning** — floating elements (popover, dropdown, tooltip, select) use a positioning engine (built on Floating UI) for collision detection, flipping, and side/align offsets.

Nothing here is styled, and almost nothing is left to the consumer that is accessibility-critical. The library's job is to be the correct-behavior substrate; your job is everything visual.

## Production Notes

**The many-packages problem, mostly solved.** For years Radix shipped as dozens of independent `@radix-ui/react-*` packages, each version-bumped separately. Real apps accumulated version drift and duplicated shared internals in the bundle. The unified `radix-ui` package (2024) is the recommended fix — one dependency, one version line, namespaced imports. Mixing old per-component packages with the unified one in the same tree can still duplicate code; standardize on one.

**Portalled content escapes your CSS tree.** `Portal` renders `Content` at the document body by default. That is correct for stacking and overflow, but means your content sits outside any scoped/parent styles and inherits nothing — expect to manage `z-index`, theming, and font inheritance explicitly. In apps with their own stacking-context regimes this is a recurring source of "why is my dropdown behind the header" bugs.

**Exit animations are opt-in.** Because Presence unmounts on animation end, forgetting to define a closing animation/transition means content vanishes instantly; conversely, a `display:none` on the closed state can prevent the exit animation from ever running. Style off `data-state` open/closed, not conditional rendering.

**Unstyled means real work.** Budget for CSS on every state of every component — hover, focus-visible, disabled, open/closed, checked, highlighted. This is the honest cost that shadcn/ui hides by shipping pre-written styles on top.

**Maintenance history matters.** After Modulz wound down its design tooling, the library went through a slow patch during 2022–2023 with issues and PRs accumulating. WorkOS's stewardship revived active development; the open-issue count (several hundred) is large but reflects a broad surface and long backlog rather than abandonment. Check that specific niche components you depend on are actively patched.

**SSR and React version.** Radix relies on `useId` and works cleanly with React 18+ hydration; it now supports React 19. Peer deps target `react`/`react-dom`, so mismatched React majors in a monorepo surface as duplicate-context runtime errors.

## When to Use / When Not

**Use when:**
- You are building or already own a design system and want correct accessibility/behavior without hand-writing focus traps and ARIA.
- You want full visual control and are willing to write the CSS.
- You are using shadcn/ui or Tailwind and want the behavioral layer underneath.

**Avoid when:**
- You want components that look finished out of the box — reach for Radix Themes, MUI, Mantine, or Chakra instead.
- Your team will not invest in styling every interactive state.
- You need a broad set of accessible *hooks* rather than fixed component structures (React Aria fits that shape better).

## Alternatives

- ariakit/ariakit — lower-level and broader primitive set with a different composition model; use when you need finer control than Radix's fixed compound structures.
- adobe/react-spectrum — React Aria's accessibility hooks are more comprehensive and framework-shaped; use when you want a11y behavior decoupled from any prescribed DOM.
- tailwindlabs/headlessui — smaller component set from the Tailwind team; use when you only need a few widgets and want a lighter dependency.
- mui/base-ui — unstyled primitives effort involving former Radix contributors; use when evaluating the newer generation of headless React components.
- shadcn-ui/ui — not a competitor but the styled copy-paste layer built on Radix; use when you want Radix behavior with styling already written.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-06 | Repo created by Modulz; early unstyled primitives preview[^1]. |
| 1.0 line | 2022–2023 | Per-component `@radix-ui/react-*` packages reach 1.0 individually. |
| — | 2022–2023 | Post-Modulz maintenance slowdown; backlog accumulates. |
| — | 2024 | Unified `radix-ui` package introduced; WorkOS-led active development[^1]. |
| — | 2024–2026 | React 19 support; ongoing per-component releases. |

## References

[^1]: Repository description and README — "Maintained by @workos", "Licensed under the MIT License, Copyright © 2022-present WorkOS". https://github.com/radix-ui/primitives
[^2]: Radix Primitives documentation. https://www.radix-ui.com/primitives/docs
[^3]: Radix Primitives releases / changelog. https://www.radix-ui.com/primitives/docs/overview/releases

## Tags

react, typescript, ui-components, accessibility, headless-ui, component-library, design-systems, unstyled, wai-aria, frontend
