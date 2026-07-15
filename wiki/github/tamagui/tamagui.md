# tamagui/tamagui

> A universal style system for React and React Native, with an optimizing compiler that flattens styled components to atomic CSS on web and hoisted style objects on native.

[GitHub repo](https://github.com/tamagui/tamagui) ·
[Official website](https://tamagui.dev) ·
[License: MIT](https://github.com/tamagui/tamagui/blob/main/LICENSE)

## Overview

Tamagui is a set of libraries by Nate Wienert for writing UI once and running it across React DOM and React Native without giving up performance[^1]. It occupies the same niche as styled-components or Stitches, but its defining bet is a build-time **optimizing compiler** (`@tamagui/static`) that partially evaluates `styled()` calls, inlines theme/token lookups, and — where it can prove the styles are static — collapses a component into a plain `div` plus atomic CSS on web, or a `View` with hoisted style objects on native.

The project is really three layers that are often conflated: `@tamagui/core` (the runtime style engine and `styled()` API, zero UI components), `@tamagui/static` (the compiler), and `tamagui` (an opinionated UI kit built on core — Stacks, Button, Sheet, Dialog, etc.). You can adopt only `core` and get the cross-platform styling system without the component library, or take the whole stack.

The central tension is complexity for performance. Tamagui asks you to define a typed design system up front (`createTamagui` with tokens, themes, fonts, media queries, shorthands) and wire a bundler plugin for every target (Metro, Vite, webpack, Next). In exchange the compiler can flatten most components. When it *can't* prove a style is static — dynamic values, cross-module indirection it fails to trace — it silently falls back to the runtime path, so the performance win is real but uneven and hard to observe without inspecting output. That, plus heavy use of advanced TypeScript, is where most of the friction lives.

## Getting Started

```bash
npm create tamagui@latest   # scaffolds a starter (simple example → production monorepo)
# or add the style engine to an existing app:
npm install @tamagui/core
```

```tsx
import { createTamagui, TamaguiProvider, styled, View } from '@tamagui/core'
import { defaultConfig } from '@tamagui/config/v4'

const config = createTamagui(defaultConfig)
type Conf = typeof config
declare module '@tamagui/core' {
  interface TamaguiCustomConfig extends Conf {}
}

const Circle = styled(View, {
  backgroundColor: '$background',
  borderRadius: 100,
  width: 50,
  height: 50,
  hoverStyle: { backgroundColor: '$backgroundHover' },
  variants: {
    active: { true: { backgroundColor: '$blue10' } },
  } as const,
})

export default function App() {
  return (
    <TamaguiProvider config={config}>
      <Circle active />
    </TamaguiProvider>
  )
}
```

The runtime works on its own; the compiler is an optional bundler plugin you add once the app is running (`@tamagui/vite-plugin`, `@tamagui/next-plugin`, `@tamagui/metro-plugin`, `tamagui-loader` for webpack). It is designed for gradual adoption.

## Architecture / How It Works

**Configuration is the schema.** `createTamagui()` takes tokens (spacing, color, size, radius, zIndex), `themes` (named theme objects, including nested sub-themes like `dark_blue`), `fonts`, `media` (named breakpoints), and `shorthands` (e.g. `bg` → `backgroundColor`). Everything downstream — the `$`-prefixed values in `styled()`, the props that autocomplete, the CSS variables emitted — derives from this object. The config type is threaded through the whole app via TypeScript module augmentation, which is why type inference cost scales with theme/token size.

**`styled()` and variants.** Components are declared with a static style object plus a `variants` map (`as const` is required for inference). Variants can be boolean, enumerated, or spread functions. Inline style props (`<View padding="$4" hoverStyle={{...}}>`) are first-class and also compiler targets.

**The compiler (`@tamagui/static`).** It parses each file, finds Tamagui components, and does a partial evaluation to resolve token/theme references and variant selection at build time. Static components flatten to a host element with a generated `className` referencing atomic CSS rules; the runtime style logic is removed. Anything it can't statically resolve is left as a runtime `styled` component. Flattening rate is a real, measured metric — the README cites 49 of ~55 inline components on the homepage flattening, and a ~15% Lighthouse gain with the compiler enabled[^1].

**Cross-platform primitives.** On web, Tamagui builds on `react-native-web` semantics but emits CSS instead of inline styles. On native it produces React Native `View`/`Text` with pre-computed style objects. Media queries become real CSS `@media` on web and JS-driven `useMedia()` subscriptions on native.

**Animations are pluggable drivers.** You pick one: `@tamagui/animations-css` (web, CSS transitions/keyframes), `@tamagui/animations-react-native` (Animated), or `@tamagui/animations-moti` / `-reanimated` (Reanimated). The `animation` prop is uniform; the driver decides the implementation. Drivers are not interchangeable at runtime — the choice is part of config.

## Production Notes

**TypeScript performance is the recurring complaint.** Large theme and token sets produce large union types; `tsc` and editor responsiveness degrade in big design systems. Mitigations people use: keep theme count reasonable, split the config, and avoid over-nesting sub-themes. This is intrinsic to the "config as types" design, not a bug that gets fixed.

**Compiler output is not guaranteed.** The performance story depends on flattening, but the compiler falls back to runtime whenever it cannot statically evaluate a style (dynamic props derived from state, values imported through indirection it doesn't follow, some spread patterns). The app still works, just slower, and there is no build error — you have to inspect compiled output or the `logTimings`/debug flags to know your flattening rate. Treat compiler wins as something to measure, not assume.

**Setup is involved and bundler-specific.** Each target (Next, Vite, Expo/Metro, webpack) needs its own plugin plus a shared `tamagui.config.ts` and the module-augmentation boilerplate. Misconfiguration commonly shows up as unstyled components, missing themes on first paint, or the compiler silently doing nothing. SSR needs `Tamagui.getCSS()` wired into the document head to avoid flash-of-unstyled-content.

**Version churn.** Tamagui moves fast and has shipped breaking changes across major versions (config format has revised through `@tamagui/config` v2/v3/v4; APIs and defaults shift between minors). Pin versions across the monorepo — mismatched `@tamagui/*` package versions are a frequent source of subtle style bugs. Read release notes before upgrading; migration steps are non-trivial.

**Native footprint and ecosystem lock-in.** The full `tamagui` kit and animation drivers add dependency and bundle weight; audit what you import if native startup matters. Because your design system is expressed in Tamagui's config and `styled()` API, migrating off it later is a rewrite of the styling layer, not a swap.

**Docs and AI assistance.** Documentation is thorough in places and thin in others, especially around compiler internals and edge cases; the community leans on Discord and a hosted "Tamagui Guru" Q&A. Budget time for reading source when you hit the boundaries.

## When to Use / When Not

**Use when:**
- You ship the same UI to web and React Native / Expo and want one styling system with real code sharing.
- You want a typed design-token/theme system with light/dark and sub-themes built in.
- Web performance matters and you'll actually configure the compiler and measure flattening.
- You want an optional component kit (Sheet, Dialog, Popover, Adapt) that works on both platforms.

**Avoid when:**
- You're web-only — plain CSS, Tailwind, or a lighter CSS-in-JS gets you there with far less setup.
- You're native-only and prefer the RN idiom — NativeWind or Unistyles are simpler.
- You want minimal build config and fast `tsc`; Tamagui's compiler and type machinery are a standing cost.
- The team won't invest in learning the config/compiler model — half-configured Tamagui gives you the complexity without the payoff.

## Alternatives

- nativewind/nativewind — Tailwind utility classes for React Native + web; simpler mental model, no design-system config, use when you want Tailwind ergonomics over a compiler.
- Software-Mansion/react-native-unistyles — fast RN-first styling with a C++ core; use when you're native-led and want performance without Tamagui's web/compiler surface.
- necolas/react-native-web — the lower-level primitive Tamagui builds on for web; use directly when you want RN-on-web without an opinionated style system.
- gluestack/gluestack-ui — universal component library with a utility/token approach; use when you want components more than a styling compiler.
- facebook/stylex — Meta's build-time atomic CSS-in-JS; use when you're web-focused and want compile-time atomic CSS without cross-native scope.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2020-10 | Repo created; early universal-styling experiments[^2]. |
| 1.0 | 2023 | First stable major; `@tamagui/core` + compiler + `tamagui` kit consolidated[^3]. |
| config v3 | 2023–2024 | Revised default config / token conventions. |
| config v4 | 2024–2025 | Current default config generation (`@tamagui/config/v4`). |

## References

[^1]: Tamagui README and homepage — compiler flattening and Lighthouse figures. https://github.com/tamagui/tamagui and https://tamagui.dev/docs/intro/introduction
[^2]: GitHub repository metadata — created 2020-10-16, MIT license. https://github.com/tamagui/tamagui
[^3]: Tamagui compiler and core docs. https://tamagui.dev/docs/intro/compiler

## Tags

typescript, react, react-native, css-in-js, atomic-css, optimizing-compiler, design-system, ui-components, cross-platform, styling, expo
