# ariakit/ariakit

> Low-level, composition-first React primitives that implement the WAI-ARIA authoring practices so you don't have to.

[GitHub repo](https://github.com/ariakit/ariakit) ·
[Official website](https://ariakit.com) ·
License: MIT (packages) — see note[^1]

## Overview

Ariakit is a toolkit of unstyled, accessible React components and hooks. It is the successor to Reakit, a project the same author (Diego Haz) started in 2017; the rename to Ariakit accompanied a ground-up rewrite of the state and composition model[^2]. The goal is narrow and consistent: give you components (`Dialog`, `Combobox`, `Menu`, `Select`, `Tab`, `Popover`, and so on) that already implement the keyboard interaction, focus management, and ARIA attribute wiring described by the WAI-ARIA Authoring Practices Guide, and stay out of the way on everything else — styling, data, layout.

The defining design decision is composition. Ariakit does not hand you closed, prop-configured widgets; it hands you open building blocks. Every component renders a "role" element you can re-target with a `render` prop, and behavior is driven by explicit stores you create and pass in. This buys extreme flexibility — you can splice your own markup into any part of a menu or combobox — at the cost of more boilerplate than higher-level libraries, and a smaller, sharper API surface you have to learn.

The main tension is maturity signaling. Ariakit is used in production by real teams and is carefully engineered, but the primary package `@ariakit/react` has spent years in the `0.x` range and has never cut a `1.0`[^3]. It is stable in practice but reserves the right to make breaking changes in minor releases, which is a real planning consideration for teams that gate dependencies on semver stability.

## Getting Started

```bash
npm install @ariakit/react
```

```tsx
import { Button, Dialog, DialogDismiss, useDialogStore } from "@ariakit/react";

function Example() {
  const dialog = useDialogStore();
  return (
    <>
      <Button onClick={dialog.toggle}>Open dialog</Button>
      <Dialog store={dialog} className="dialog">
        <h2>Welcome</h2>
        <p>Focus is trapped and restored, Escape closes, ARIA is wired.</p>
        <DialogDismiss>OK</DialogDismiss>
      </Dialog>
    </>
  );
}
```

Ariakit ships no styles by default — the `className` above is yours to define. The `render` prop swaps the underlying element while keeping all behavior:

```tsx
import { Button } from "@ariakit/react";
import { Link } from "react-router-dom";

// Renders an <a> with Button's keyboard/ARIA behavior
<Button render={<Link to="/next" />}>Go</Button>
```

## Architecture / How It Works

Ariakit splits every widget into two layers: **stores** and **components**.

- **Stores** (`useDialogStore`, `useComboboxStore`, `useMenuStore`, …) hold the widget's state and expose imperative methods (`show`, `hide`, `setValue`, `move`) plus a subscribable state object. Under the hood these are built on a small reactive store abstraction rather than React context or reducers, so a single store can drive many components and state can be read/written outside render[^4].
- **Components** are thin, mostly-stateless renderers that subscribe to a store passed via the `store` prop. Because state lives outside the component tree, you can hoist a store, share it between siblings, or read it in an event handler without prop-drilling.

Composition is handled by the `render` prop, which every component accepts. Passing `render={<MyElement />}` merges Ariakit's props (event handlers, ARIA attributes, refs) into your element instead of Ariakit's default tag. This replaced Reakit's older `as` prop and is the mechanism behind Ariakit's "any component can become any element" flexibility. Prop merging is order-aware, so your handlers compose with Ariakit's rather than clobbering them.

Focus management is the load-bearing internal system: roving tabindex for composite widgets (menus, toolbars, tab lists), focus trapping and restoration for dialogs and popovers, and typeahead for listboxes. Popover positioning is delegated to Floating UI[^5]. Much of the value of the library is in the accumulated edge-case handling here — virtual focus, portalled content, nested dialogs, screen-reader quirks — which is exactly the surface most teams get wrong hand-rolling their own.

Beyond `@ariakit/react`, the monorepo ships two experimental packages: `@ariakit/solid` (a SolidJS port) and `@ariakit/tailwind` (Tailwind styling helpers). Both are explicitly experimental and should not be treated as production-stable.

## Production Notes

**Pre-1.0 versioning is the biggest operational caveat.** `@ariakit/react` publishes breaking changes in minor `0.x` bumps. Pin an exact version or a tight range and read the changelog before upgrading; do not assume `^0.4` is non-breaking the way `^1` would be.

**You bring the styling, always.** There is no default stylesheet. This is intentional and is why Ariakit composes cleanly with Tailwind, CSS Modules, or vanilla CSS — but it means "install and get a good-looking dialog" is not the experience. Budget design time. The Ariakit website ships a large set of copyable styled examples that fill this gap; some (the "Plus" examples) are under a separate paid license[^1].

**Stores are powerful and easy to misuse.** Creating a store on every render, or passing an unstable store between components, causes subtle state desync. Follow the documented pattern: create the store with the `use*Store` hook at the right level and pass it down, or let child components read the store from context when nested.

**Bundle size scales with what you import.** The package is tree-shakeable and per-component, so cost tracks usage, but composite widgets (combobox, select, menu) pull in Floating UI and the focus machinery and are not tiny. Measure if you only need one small primitive.

**Accessibility is a strong default, not an automatic guarantee.** Ariakit wires the ARIA roles and keyboard behavior, but you still own accessible names, labels, contrast, and correct semantic nesting. Test with a real screen reader; the library gets you most of the way, not all of it.

## When to Use / When Not

**Use when:**
- You need accessible widgets (dialog, combobox, menu, select) but want full control over markup and styles.
- You're building a design system and want unopinionated, composable primitives to build on.
- You value fine-grained state control — stores you can read and drive imperatively.

**Avoid when:**
- You want styled, batteries-included components out of the box (reach for MUI, Chakra, or Mantine).
- Your team needs strict semver stability and can't absorb breaking changes in minor releases.
- You only need one or two simple elements and don't want to learn the store/composition model.

## Alternatives

- radix-ui/primitives — similar unstyled/accessible goal with a more conventional prop-driven API; larger community and the base for shadcn/ui. Use when you prefer closed components over Ariakit's open composition.
- adobe/react-spectrum — React Aria hooks give lower-level accessibility behavior with no rendering at all. Use when you want to own the DOM entirely and only borrow behavior.
- tailwindlabs/headlessui — smaller set of unstyled components tuned for Tailwind. Use for a lighter dependency when you only need a few widgets.
- mui/base-ui — newer unstyled primitives from the MUI team (with Radix contributors). Use when evaluating the current unstyled-component landscape.
- shadcn-ui/ui — copy-paste styled components built on Radix. Use when you want to own the source and skip a runtime dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| Reakit | 2017 | Original project by Diego Haz; hook-and-`as`-prop model[^2]. |
| Rename to Ariakit | ~2022 | Rebrand plus rewrite toward the store + `render` composition model[^2]. |
| @ariakit/react 0.2–0.4 | 2023–2025 | Store-based API stabilized in practice; Floating UI for positioning; Solid and Tailwind packages added as experimental[^3]. |

## References

[^1]: The monorepo is multi-licensed. The `packages` directory (including `@ariakit/react`) is MIT; the docs `app` is proprietary and some "Plus" examples use the Ariakit Plus License. GitHub reports no single top-level SPDX license as a result. https://github.com/ariakit/ariakit#licensing
[^2]: Ariakit is the successor to Reakit, both by Diego Haz. https://ariakit.com and https://haz.dev
[^3]: `@ariakit/react` on npm — versioning has remained in the 0.x range. https://www.npmjs.com/package/@ariakit/react
[^4]: Ariakit documentation, "Component stores". https://ariakit.com/guide/component-stores
[^5]: Floating UI — positioning engine used for popovers, menus, and tooltips. https://floating-ui.com

## Tags

react, typescript, accessibility, wai-aria, ui-components, headless-ui, design-system, unstyled, a11y, frontend
