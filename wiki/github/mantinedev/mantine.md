# mantinedev/mantine

> A TypeScript-first React component library that bet on native CSS modules over CSS-in-JS.

[GitHub repo](https://github.com/mantinedev/mantine) ·
[Official website](https://mantine.dev) ·
[License: MIT](https://github.com/mantinedev/mantine/blob/master/LICENSE)

## Overview

Mantine is a React component and hooks library created by Vitaly Rtishchev, first published in 2021[^1]. It ships as a set of scoped npm packages — `@mantine/core` (100+ components), `@mantine/hooks` (80+ standalone hooks), plus optional packages for forms, charts, notifications, a command palette (`spotlight`), a Tiptap-based rich text editor, and more. It competes in the same space as MUI, Chakra UI, and Ant Design: a batteries-included component kit for teams that want accessible, themeable UI without assembling primitives themselves.

Mantine's defining decision is its styling engine. Through version 6 it used Emotion (CSS-in-JS) with a `createStyles` API and an `sx` prop. Version 7 (2023) removed Emotion entirely and rebuilt every component on native CSS Modules with CSS custom properties for theming[^2]. This traded runtime flexibility for smaller bundles, faster rendering, and clean React Server Component compatibility — at the cost of a hard, ecosystem-wide migration. Any assessment of Mantine hinges on which side of the v6→v7 line a project sits.

The library is actively maintained: the default branch sees near-daily pushes, the issue backlog is kept small (tens, not thousands), and releases are frequent. `@mantine/hooks` is widely used on its own, independent of the component set.

## Getting Started

```bash
npm install @mantine/core @mantine/hooks
```

```tsx
import "@mantine/core/styles.css";           // required, global CSS
import { MantineProvider, Button } from "@mantine/core";

export default function App() {
  return (
    <MantineProvider>
      <Button variant="filled" onClick={() => alert("hi")}>
        Click me
      </Button>
    </MantineProvider>
  );
}
```

The `@mantine/core/styles.css` import is mandatory — components are unstyled without it. Every app must be wrapped in `MantineProvider`, which injects the theme's CSS variables. Package-specific stylesheets (e.g. `@mantine/dates/styles.css`) are imported separately, only for the packages you use.

## Architecture / How It Works

Since v7, styling works through three cooperating layers:

- **CSS Modules** define each component's base styles at build time. There is no runtime style serialization; the CSS ships as static files.
- **CSS custom properties** carry theme values. `MantineProvider` resolves the `theme` object (colors, spacing, radii, fonts) into `--mantine-*` variables on a root element, so theme changes are a variable swap, not a re-render.
- **`data-*` attributes** express component state and variants (`data-variant`, `data-disabled`, `data-position`), which CSS Modules select against. This replaced the old prop-driven Emotion styles.

Component-level customization uses the `classNames` and `styles` props to target internal "elements" (each component documents its named parts, e.g. `Button` exposes `root`, `label`, `inner`, `section`). Style props (`mt`, `c`, `bg`, etc.) remain as shortcuts that map to inline CSS variables. The removed pieces are `createStyles`, the `sx` prop's dynamic form, and the global Emotion cache.

Because there is no CSS-in-JS runtime, most Mantine components are compatible with React Server Components, though they still render on the client — nearly all carry `"use client"`. Dark mode is handled by a `data-mantine-color-scheme` attribute on `<html>`, toggled by `useMantineColorScheme`, with values persisted to `localStorage`.

Mantine emits its CSS inside an `@layer mantine` cascade layer, which lets application and utility-framework styles override component styles predictably regardless of import order[^3].

## Production Notes

**The v6→v7 migration is the biggest operator cost.** Emotion, `createStyles`, and the dynamic `sx` prop were removed, so any codebase that styled components through them must be rewritten to CSS Modules, `classNames`, or plain `style`. There is no automated codemod that fully covers this; teams on heavily-customized v6 apps have reported multi-week migrations. Mantine keeps v6 docs online, but v6 is not receiving feature work.

**CSS import order and cascade layers.** When combining Mantine with Tailwind or other global CSS, specificity conflicts are common. Mantine's `@layer mantine` helps, but Tailwind v3 does not emit its utilities into a layer by default, so unlayered Tailwind can unexpectedly win or lose against Mantine depending on setup. Verify the generated cascade order explicitly rather than assuming.

**Next.js App Router setup is non-trivial.** To avoid a color-scheme flash on load you must render `ColorSchemeScript` in `<head>` and wrap the tree in `MantineProvider`; without it, dark-mode users see a light flash on first paint. Mantine documents a specific `layout.tsx` recipe for this[^4]. Components are client components, so pushing interactivity to leaves matters for RSC payload size.

**Bundle and tree-shaking.** JavaScript tree-shakes per-import, but the stylesheet is global and shipped whole per package — you pay for `@mantine/core/styles.css` even if you use a handful of components. For strict CSS budgets this is a consideration; there is no official per-component CSS splitting.

**Version churn.** Minor releases are frequent and occasionally adjust component internals (named element keys, `data-*` names) that custom `classNames`/`styles` overrides depend on. Pin versions and re-test heavily-restyled components on upgrades.

## When to Use / When Not

**Use when:**
- You want a large, accessible, TypeScript-native component set with a coherent theme system and no Material Design lock-in.
- You value the standalone `@mantine/hooks` collection and a forms library (`@mantine/form`) from the same author.
- You want RSC-compatible components and static CSS rather than a CSS-in-JS runtime.
- You're starting fresh (v7/v8) rather than maintaining a large v6 codebase.

**Avoid when:**
- You have a large existing v6 app and cannot budget the CSS-in-JS removal migration.
- You need a specific mature design language out of the box (Material → MUI, enterprise data tables → Ant Design).
- You want to own component source and style purely with Tailwind — a copy-paste approach fits better.
- Your CSS budget forbids shipping a library's global stylesheet.

## Alternatives

- mui/material-ui — use instead when you need Google Material Design and the largest component/enterprise ecosystem.
- chakra-ui/chakra-ui — use instead when you prefer Chakra's style-props ergonomics and don't mind its own styling-engine trajectory.
- shadcn-ui/ui — use instead when you want to own the component code and style everything with Tailwind + Radix primitives.
- radix-ui/primitives — use instead when you want unstyled, headless, accessibility-focused primitives and full control of styling.
- ant-design/ant-design — use instead for data-dense enterprise admin UIs with rich tables, forms, and built-in i18n.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2021-05 | First stable release; core components + hooks[^1]. |
| 5.0 | 2022-08 | Expanded component set, theming improvements. |
| 6.0 | 2023-02 | Last major version built on Emotion / `createStyles`. |
| 7.0 | 2023-09 | Emotion removed; rebuilt on CSS Modules + CSS variables[^2]. |
| 8.0 | 2025 | Continued CSS-Modules architecture; new components and APIs. |

## References

[^1]: Mantine repository and package history, mantinedev/mantine. https://github.com/mantinedev/mantine
[^2]: Mantine 7.0.0 release — migration from CSS-in-JS to native CSS. https://mantine.dev/changelog/7-0-0/
[^3]: Mantine docs, "CSS layers" and styles overview. https://mantine.dev/styles/mantine-styles/
[^4]: Mantine docs, "Usage with Next.js" (App Router setup, ColorSchemeScript). https://mantine.dev/guides/next/

## Tags

react, typescript, component-library, ui-kit, css-modules, hooks, design-system, dark-mode, frontend, theming
