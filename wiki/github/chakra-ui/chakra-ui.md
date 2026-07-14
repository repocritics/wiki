# chakra-ui/chakra-ui

> Accessible React component system built on headless state machines and a runtime CSS-in-JS styling engine.

[GitHub repo](https://github.com/chakra-ui/chakra-ui) ·
[Official website](https://chakra-ui.com) ·
[License: MIT](https://github.com/chakra-ui/chakra-ui/blob/main/LICENSE)

## Overview

Chakra UI is a React component library created by Segun Adebayo, first published in 2019[^1]. It ships prebuilt, WAI-ARIA-conformant components (buttons, menus, dialogs, form controls, layout primitives) with a token-based theme and inline "style props" — you set `padding`, `bg`, or `gap` directly as JSX props rather than writing CSS. Its historical selling point was that accessibility and theming came correct by default, which made it a common default for internal tools, dashboards, and SaaS admin surfaces.

The project has two materially different eras. Versions 1 and 2 (2021–2022) were built on Emotion plus Styled System, exposed as a large fleet of `@chakra-ui/*` packages and configured through `extendTheme`. Version 3 (2024) is a ground-up rewrite: a single `@chakra-ui/react` package, interactive component logic delegated to Ark UI / Zag.js state machines, and a new recipe-based styling system inspired by Panda CSS[^2]. Public component APIs, prop names, and the theming entry points all changed, so v2 knowledge does not transfer cleanly to v3.

The defining tension is styling strategy. Chakra remains **runtime** CSS-in-JS (Emotion), computing styles in the browser. That buys ergonomic style props and dynamic theming but pays a bundle-size and runtime cost, and it constrains how components behave under React Server Components — a tradeoff that zero-runtime peers (Tailwind, Panda CSS, vanilla-extract) do not carry.

## Getting Started

```sh
npm i @chakra-ui/react @emotion/react
```

v3 requires a provider wrapping the app, supplied with a styling "system":

```tsx
// app/provider.tsx
"use client"
import { ChakraProvider, defaultSystem } from "@chakra-ui/react"

export function Provider({ children }: { children: React.ReactNode }) {
  return <ChakraProvider value={defaultSystem}>{children}</ChakraProvider>
}
```

```tsx
import { Button, HStack } from "@chakra-ui/react"

export default function App() {
  return (
    <HStack gap="4">
      <Button colorPalette="teal">Save</Button>
      <Button variant="outline">Cancel</Button>
    </HStack>
  )
}
```

## Architecture / How It Works

Chakra v3 splits into two layers that were previously entangled:

1. **Behavior** — interactive components (menu, dialog, combobox, tooltip, slider, etc.) are thin React wrappers over Ark UI[^3], which is itself powered by Zag.js[^4]: framework-agnostic finite state machines that model focus, keyboard interaction, and ARIA wiring. This is where the accessibility guarantees actually live, and it is shared with the Vue/Solid ports of Ark.
2. **Styling** — Emotion computes CSS at runtime from a token system. v3 replaces v2's `extendTheme` merge model with `createSystem(defaultConfig, customConfig)`, which produces the system object handed to `ChakraProvider`. Component appearance is expressed through **recipes** (`defineRecipe`) and **slot recipes** (`defineSlotRecipe`) for multi-part components — a variant/size matrix compiled into class-driven styles, closely mirroring Panda CSS's model.

Style props on any component are parsed against the theme's tokens, conditions (hover, dark mode, breakpoints), and semantic tokens, then emitted as Emotion styles. Dark mode is no longer built in: v3 delegates color-mode handling to `next-themes` rather than a bundled `ColorModeProvider`.

v3 also adopts a **snippet** model for composite UI. Rather than importing an opinionated `<Modal>`, you generate local, editable composition components (e.g. via the Chakra CLI's `snippet add`), copying source into your repo in the style popularized by shadcn/ui. The intent is that high-level composition is yours to own while the primitives stay in the package.

## Production Notes

**RSC and the "use client" boundary.** Chakra components are client components. In Next.js App Router they must sit under a client-side provider, and the provider file needs `"use client"`. Chakra advertises Next.js RSC compatibility, but in practice that means "usable within a client subtree," not "renders as a server component." Emotion's runtime still requires SSR style extraction to avoid a flash of unstyled content.

**Runtime cost.** Because styling is computed in the browser, style-prop-heavy trees add measurable CPU and bundle overhead versus zero-runtime systems. For large tables/lists rendering thousands of styled nodes, this shows up in profiles; memoization and pulling repeated styling into recipes helps more than micro-optimizing props.

**The v2 → v3 migration is a rewrite, not a bump.** Package consolidation (many `@chakra-ui/*` → one), prop renames (`spacing` → `gap`, `isDisabled` → `disabled`, `isOpen` → `open`), the `extendTheme` → `createSystem` change, removal/relocation of some components, and the move to snippet-based composition all land together[^5]. Teams have reported it as effectively re-adopting the library. There is a migration guide and a codemod, but expect manual work.

**Dropped peer dependencies.** v3 no longer requires `framer-motion` or `@emotion/styled` — the install is just `@chakra-ui/react @emotion/react`. Animation-dependent v2 code and any direct `styled()` usage must be reworked.

**Maintenance shape.** The repo is actively maintained (last push 2026-07-14) with a low open-issue count, but development is concentrated around one primary author. Much of the accessibility surface now lives upstream in Ark UI / Zag.js, so bugs in interactive behavior often trace to those repositories rather than this one.

## When to Use / When Not

**Use when:**
- You want accessible, batteries-included React components without hand-wiring ARIA and focus management.
- You value inline style props and a token/theme system for consistent design across an app.
- You are building dashboards, internal tools, or SaaS UIs where developer velocity matters more than shaving every kilobyte.

**Avoid when:**
- You need zero-runtime styling or a fully server-rendered (RSC-first) component tree — a runtime CSS-in-JS library fights that goal.
- You are on Chakra v2 and cannot budget a substantial migration; adopt v3 deliberately, not incidentally.
- You want unstyled primitives to fully own the design layer — Radix or Ark UI directly are a closer fit.

## Alternatives

- mui/material-ui — heavier, Material-Design-opinionated, largest ecosystem; use when you want a complete design language out of the box.
- radix-ui/primitives — unstyled accessible primitives; use when you want to own 100% of styling with no runtime CSS engine.
- shadcn-ui/ui — copy-in components over Radix + Tailwind; use when you want zero-runtime styling and full source ownership.
- mantinedev/mantine — similar hooks-plus-components scope with its own styling approach; use as a close feature-for-feature Chakra alternative.
- chakra-ui/ark — the headless layer underneath Chakra v3; use when you want the state machines without Chakra's styling opinions.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2019 | Initial release by Segun Adebayo. Emotion + Styled System foundation.[^1] |
| 1.0 | 2021 | First stable major; theming via `extendTheme`, multi-package layout. |
| 2.0 | 2022-06 | Emotion/Styled-System era matured; React 18 support, many `@chakra-ui/*` packages. |
| 3.0 | 2024 | Rewrite: single package, Ark UI / Zag.js behavior layer, recipe styling, snippet composition, `createSystem`.[^2] |

## References

[^1]: Chakra UI — official site and documentation. https://chakra-ui.com
[^2]: Chakra UI v3 announcement. https://chakra-ui.com/blog/00-announcing-v3
[^3]: Ark UI — headless component library used by Chakra v3. https://ark-ui.com
[^4]: Zag.js — finite state machines for UI components. https://zagjs.com
[^5]: Chakra UI v3 migration guide. https://chakra-ui.com/docs/get-started/migration

## Tags

react, typescript, component-library, design-system, css-in-js, accessibility, ui-components, emotion, ark-ui, react-components, wai-aria
