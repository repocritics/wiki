# callstack/react-native-paper

> Material Design component library for React Native — a themeable UI kit for Android and iOS from a single codebase.

[GitHub repo](https://github.com/callstack/react-native-paper) ·
[Official website](https://reactnativepaper.com) ·
[License: MIT](https://github.com/callstack/react-native-paper/blob/main/LICENSE.md)

## Overview

React Native Paper is a UI component library maintained by Callstack that implements Google's Material Design for React Native[^1]. It provides ~40 pre-built, themeable components — buttons, cards, app bars, text inputs, dialogs, snackbars, FABs, data tables, chips — that render as native views on both Android and iOS. It is one of the oldest and most-adopted React Native component kits, with roughly 14k GitHub stars as of 2026 and steady maintenance (last pushed mid-2026).

The defining tension is that Paper is opinionated about *looking like Material Design*. On Android that matches platform convention; on iOS it produces an interface that follows Google's design language rather than Apple's Human Interface Guidelines. Paper does apply some platform adaptations, but an app built with it reads as "Material" on both targets. Teams that want a native-iOS look, or a fully custom design system, will spend most of their time fighting the defaults — at which point a lower-level primitive library is usually a better fit.

Since version 5 (2022) Paper targets **Material Design 3** ("Material You")[^2]. The MD3 theme is the default; the older Material Design 2 themes remain available for apps that have not migrated. This version boundary is the single most important thing to know before adopting: v4 and v5 have different theming APIs, different default component appearances, and different color-token structures.

## Getting Started

```bash
npm install react-native-paper react-native-safe-area-context react-native-vector-icons
# Expo:
npx expo install react-native-paper react-native-safe-area-context @expo/vector-icons
```

Wrap the app in `PaperProvider` and consume the theme:

```tsx
import * as React from 'react';
import { PaperProvider, Button, Card, Text } from 'react-native-paper';

export default function App() {
  return (
    <PaperProvider>
      <Card style={{ margin: 16 }}>
        <Card.Title title="Paper" subtitle="Material Design 3" />
        <Card.Content>
          <Text variant="bodyMedium">Themeable RN components.</Text>
        </Card.Content>
        <Card.Actions>
          <Button mode="contained" onPress={() => {}}>OK</Button>
        </Card.Actions>
      </Card>
    </PaperProvider>
  );
}
```

Icons come from `react-native-vector-icons` (MaterialCommunityIcons by default). On bare React Native the font must be linked/bundled or icons render as empty boxes.

## Architecture / How It Works

Paper is a **theme + component** library layered on React Native's core primitives (`View`, `Text`, `Pressable`, `Animated`). It does not ship its own rendering engine — every component is a composition of RN views plus styling derived from a theme context.

- **Theme context.** `PaperProvider` puts a theme object into React context. Components read it via the `useTheme` hook or the `withTheme` HOC. The theme carries a color token set, typography scale (`variant` props like `bodyMedium`, `titleLarge`), roundness, and elevation. `MD3LightTheme` / `MD3DarkTheme` are the v5 defaults; `MD2LightTheme` / `MD2DarkTheme` remain for backward compatibility, and the theme's `version` field (2 or 3) switches component rendering paths internally.
- **Color system.** MD3 themes expose the full Material You token palette (`primary`, `onPrimary`, `primaryContainer`, `surface`, `surfaceVariant`, `elevation.level0…5`, etc.). Custom themes are built by overriding these tokens; Material's own theme-builder output can be dropped in.
- **Icons.** Components that render icons accept an `icon` prop resolved through a configurable icon provider — by default `react-native-vector-icons/MaterialCommunityIcons`. This is a peer dependency, not bundled.
- **Navigation integration.** `adaptNavigationTheme` bridges Paper's theme to React Navigation so both share one color source, avoiding two competing theme systems.

Because everything derives from a single context object, theming is genuinely global and consistent — but it also means the theme shape is a hard API surface. The MD2→MD3 token rename in v5 is why upgrades are non-trivial: any code that read `theme.colors.accent` (MD2) has to move to the MD3 token model.

## Production Notes

**The v4 → v5 migration is the main upgrade pain.** Color tokens were renamed and restructured for MD3, several components changed default appearance, and `Provider` became `PaperProvider`. There is an official migration guide, but the change is large enough that many teams pinned v4 for a long time. Do not treat it as a routine bump.

**Icon fonts are the most common first-run bug.** On bare RN, missing font linking produces tofu/empty squares instead of icons. On Expo the managed `@expo/vector-icons` path avoids most of this; bare projects must ensure the vector-icons font is bundled for both platforms.

**Bundle size and tree-shaking.** Importing from the package root pulls the component graph. Metro's tree-shaking is limited compared to web bundlers, so app size is a real consideration on large screens' worth of components. Import only what you use and measure with a bundle analyzer if size matters.

**Performance.** Most components are thin and fine at scale, but `DataTable` is a layout-only component (not virtualized) — large datasets should use `FlatList`/`FlashList`, not `DataTable`. Heavy use of `Portal` (used by `Dialog`, `Modal`, `Snackbar`, `Menu`) requires a `Portal.Host`; nesting and ordering bugs here cause "component renders behind others" issues.

**iOS aesthetics.** Accept that the output looks Material. If the product spec demands native-iOS controls (system-style switches, action sheets, segmented controls), you will be re-skinning heavily or mixing in another library.

**RTL and accessibility** are generally handled at the RN layer; Paper components forward accessibility props, but complex custom compositions still need manual `accessibilityRole`/`accessibilityLabel` review.

## When to Use / When Not

**Use when:**
- You want a batteries-included Material Design look on Android + iOS from one codebase.
- You value a mature, documented, single-vendor-maintained component set over assembling primitives.
- Your design system is Material (or you are happy to adopt Material You tokens).
- You are on Expo and want a low-friction, well-supported UI kit.

**Avoid when:**
- You need an iOS-native look and feel, or a fully bespoke design language.
- You want maximal tree-shaking / minimal bundle and only need a handful of components.
- You need virtualized data grids or heavy list UIs (Paper's `DataTable` is not that).
- You prefer a styling-primitive approach (utility props, compile-time styling) over pre-styled components.

## Alternatives

- tamagui/tamagui — compile-time optimized styling + component kit; better tree-shaking and cross-platform (web) story, steeper setup.
- gluestack/gluestack-ui — utility-first, copy-in components and Tailwind-like styling; the successor lineage to the now-deprecated NativeBase.
- react-native-elements/react-native-elements — older, cross-platform themed toolkit; broad but less actively opinionated than Paper.
- wix/react-native-ui-lib — Wix's large component + design-token library; comprehensive but heavier.
- Use plain react-native core + a styling lib (unistyles, StyleSheet) instead when you want full control and minimal footprint.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017 | Early Material Design component set for React Native[^1]. |
| 2.0 | 2018 | Theming and component API maturation. |
| 3.0 | 2019 | Expanded component coverage, TypeScript adoption. |
| 4.0 | 2020 | Material Design 2 refinements, theming improvements. |
| 5.0 | 2022 | Material Design 3 ("Material You") default, MD3 token system, `PaperProvider`[^2]. |

## References

[^1]: React Native Paper — Material Design for React Native (Callstack). https://reactnativepaper.com
[^2]: React Native Paper docs — Material Design 3 / theming guide. https://callstack.github.io/react-native-paper/docs/guides/theming
[^3]: Callstack open source — maintainer. https://callstack.com/open-source

## Tags

react-native, material-design, ui-kit, mobile, components, typescript, android, ios, theming, expo, cross-platform
