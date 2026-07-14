# tailwindlabs/headlessui

> Completely unstyled, fully accessible UI primitives (menus, dialogs, comboboxes) that ship behavior and ARIA but zero appearance.

[GitHub repo](https://github.com/tailwindlabs/headlessui) ·
[Official website](https://headlessui.com) ·
[License: MIT](https://github.com/tailwindlabs/headlessui/blob/main/LICENSE)

## Overview

Headless UI is a set of "headless" component primitives from Tailwind Labs (the Tailwind CSS team). Each component — `Menu`, `Listbox`, `Combobox`, `Dialog`, `Popover`, `Disclosure`, `Switch`, `RadioGroup`, `Tab`, `Transition` — implements the keyboard interaction, focus management, and WAI-ARIA wiring that the WAI-ARIA Authoring Practices call for, and renders no styling of its own[^1]. You bring the markup and classes; the library brings the behavior. The name and marketing pair it with Tailwind CSS, but there is no hard dependency: the components emit plain elements and `data-*` attributes, so any styling system works.

It ships as three npm packages: `@headlessui/react`, `@headlessui/vue`, and `@headlessui/tailwindcss` (a small Tailwind plugin that exposes variants like `ui-open:` matching component state)[^2]. React and Vue are maintained from the same monorepo but are not at feature parity — the React line moved to a v2 rewrite that added components and an internal positioning engine the Vue line did not receive.

The defining tradeoff is ownership versus effort. Because nothing is styled, you never fight a design system you didn't choose and you get correct accessibility for free — but you write every wrapper, every class, and every empty/focus/selected visual state yourself. For teams that want pre-styled parts, the common path is a layer on top (shadcn/ui does this over Radix, not Headless UI); Headless UI itself deliberately stops at behavior.

## Getting Started

```bash
npm install @headlessui/react@latest
# or: npm install @headlessui/vue@latest
```

```tsx
// React — an accessible dropdown menu with zero built-in styling
import { Menu, MenuButton, MenuItems, MenuItem } from "@headlessui/react";

export function AccountMenu() {
  return (
    <Menu>
      <MenuButton className="rounded bg-gray-800 px-3 py-1 text-white">
        Options
      </MenuButton>
      <MenuItems anchor="bottom" className="rounded border bg-white shadow">
        <MenuItem>
          {/* data-focus is set on keyboard/hover focus — style it yourself */}
          <a href="/settings" className="block px-4 py-2 data-focus:bg-gray-100">
            Settings
          </a>
        </MenuItem>
        <MenuItem>
          <a href="/logout" className="block px-4 py-2 data-focus:bg-gray-100">
            Sign out
          </a>
        </MenuItem>
      </MenuItems>
    </Menu>
  );
}
```

Keyboard navigation, roving tabindex, `aria-*` attributes, outside-click dismissal, and focus return are all handled by the components. The `anchor` prop (React v2) positions `MenuItems` relative to the button via a built-in Floating UI integration.

## Architecture / How It Works

Each primitive is a small compound-component tree that coordinates through React context (or Vue provide/inject). A `Menu` root holds open/closed state and an active-item index; `MenuButton`, `MenuItems`, and `MenuItem` read that context to wire `aria-expanded`, `aria-activedescendant`, `role`, and keyboard handlers. State is exposed to your markup as **data attributes** — `data-open`, `data-closed`, `data-focus`, `data-selected`, `data-disabled` — which you target with CSS or Tailwind variants. In React v1 this state was primarily exposed through render-prop children (`{({ active }) => ...}`); v2 kept the render props but made data attributes the recommended styling surface[^3].

React v2 (2024) was a substantial rewrite[^4]. It replaced the older render-prop-heavy API with a flatter, individually importable component set (`MenuButton` instead of `Menu.Button`), pulled in **Floating UI** for anchored positioning (the `anchor` prop and `<...Items anchor="bottom start">` syntax), and added primitives that never existed in v1: `Checkbox`, `Fieldset`, `Legend`, `Field`, `Label`, `Description`, `Input`, `Select`, `Textarea`, `Button`, and `DataInteractive`. The `Combobox` gained built-in virtualization for long option lists. Transitions were reworked so that a component can animate itself via `transition` + `data-closed` CSS instead of wrapping everything in the `Transition` render-prop component.

`Dialog` is the most involved primitive: it renders through a portal, installs a focus trap, marks sibling content `aria-hidden`, restores focus to the trigger on close, and manages scroll locking. `Popover` and `Menu` share the outside-click and escape-to-close machinery. Because these components own focus and portal behavior, they are where most integration surprises live.

## Production Notes

**React v1 → v2 is a real migration, not a version bump.** Import paths change (`Menu.Button` → `MenuButton`), the positioning story changes (manual wrappers → `anchor` prop), and transition patterns change. There is an official codemod and migration guide, but mixed-version codebases and third-party wrappers built on v1 APIs need touching[^4]. Budget time; do not treat it as a patch upgrade.

**Vue lags React.** The v2 rewrite and its new components landed on `@headlessui/react` first. If you are on Vue and reading React examples from the docs or blog posts, confirm the component exists in `@headlessui/vue` before designing around it. For Vue-first teams, community libraries (Reka UI, formerly Radix Vue) often have wider coverage.

**You own every empty state.** Because nothing is styled, forgetting to style `data-focus`, `data-selected`, or the disabled state produces a component that works for a screen reader but looks broken to sighted users. Keyboard-only focus states in particular are easy to omit and hard to catch without deliberate keyboard testing.

**SSR / hydration.** `Dialog` and other portal-based components have historically produced hydration and `id`-mismatch warnings in Next.js and other SSR setups; the library uses stable ID generation (React's `useId`) to mitigate this, but portaled content plus scroll-lock still interacts with layout-shift and `overflow` handling in ways worth testing on real pages, not just in dev.

**Positioning edge cases.** The Floating UI `anchor` integration covers common placement, but constrained containers, transformed ancestors, and nested overflow scroll can still clip or mis-place floating panels — the same class of problems any popover library faces. For heavy custom positioning you may still reach for Floating UI directly.

**Scope is intentionally narrow.** There is no date picker, no data table, no toast, no tooltip in the core set. Headless UI covers a specific list of interactive patterns and stops. If your design system needs the long tail, plan to combine it with other primitives (React Aria, Radix) or build them.

## When to Use / When Not

**Use when:**
- You already use Tailwind CSS and want accessible interactive primitives that stay out of your styling.
- You want correct keyboard + ARIA behavior for menus/dialogs/comboboxes without auditing it yourself.
- You're on React and can adopt v2, or on Vue and only need the covered primitives.
- You want full visual control and are willing to style every state.

**Avoid when:**
- You want pre-styled, drop-in components — reach for a styled layer (shadcn/ui, a component kit) instead.
- You need a broad component catalog (date pickers, tables, tooltips, toasts) from one dependency.
- You're on Vue and depend on the newest primitives that only exist in the React v2 line.
- You want a framework-agnostic hook-level primitive layer rather than fixed compound components.

## Alternatives

- radix-ui/primitives — React-only headless primitives with a wider component catalog (tooltip, toast, accordion, slider); use instead when you need more parts than Headless UI covers.
- adobe/react-spectrum — React Aria hooks are the most exhaustive accessibility layer; use instead when you want low-level hooks and behavior you compose yourself rather than fixed components.
- ariakit/ariakit — lower-level, rigorously accessible React components; use instead when you want fine-grained control over composition and state.
- unovue/reka-ui — Vue port in the Radix mold (formerly Radix Vue); use instead when you're Vue-first and want broader, actively-expanded coverage.
- shadcn-ui/ui — copy-paste styled components built on Radix; use instead when you want owned source you can edit rather than an installed dependency, and don't mind not being headless.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-09 | Repository created; first primitives for React and Vue[^5]. |
| v1.0 (React) | 2021 | Stable v1 API — compound components via `Menu.Button` style, render props for state. |
| v1.7 (React) | 2022 | `Combobox` and later refinements on the v1 line. |
| v2.0 (React) | 2024 | Major rewrite: flat imports, Floating UI `anchor` prop, new form primitives, data-attribute transitions[^4]. |

Dates for intermediate releases are approximate; consult the repository's release notes for exact tags.

## References

[^1]: Headless UI documentation. https://headlessui.com
[^2]: `@headlessui/tailwindcss` package — Tailwind variant plugin for component state. https://www.npmjs.com/package/@headlessui/tailwindcss
[^3]: Headless UI React styling with data attributes / render props. https://headlessui.com/react/menu
[^4]: Tailwind Labs, "Headless UI v2.0 for React". https://tailwindcss.com/blog/headless-ui-v2
[^5]: GitHub repository metadata (created 2020-09-16). https://github.com/tailwindlabs/headlessui

## Tags

typescript, react, vue, accessibility, headless-ui, components, ui-primitives, a11y, tailwindcss, aria, frontend
