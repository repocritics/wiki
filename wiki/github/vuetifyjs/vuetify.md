# vuetifyjs/vuetify

> A Vue component framework implementing Google's Material Design — a large, opinionated UI kit that trades bundle weight and design flexibility for coverage.

[GitHub repo](https://github.com/vuetifyjs/vuetify) ·
[Official website](https://vuetifyjs.com) ·
[License: MIT](https://github.com/vuetifyjs/vuetify/blob/master/LICENSE.md)

## Overview

Vuetify is a Vue-only component library that implements the Material Design specification[^1]. It was created by John Leider in 2016 and has been maintained since by a paid core team funded through GitHub Sponsors, Open Collective, and Tidelift[^2]. Its selling point is breadth: dozens of components (data tables, date pickers, steppers, virtual scrollers, a 12-column grid, a theming system, 42+ i18n locales) that all share one Material-derived visual language, so a team can build a complete application without sourcing widgets from multiple libraries or writing much CSS.

The framework's defining tension is coupling — in two directions. First, it is bound to Material Design: components look and behave like Material, and while colors, SASS variables, and "blueprints" let you retheme, escaping the Material look entirely fights the framework rather than using it. Second, it is bound to Vue's major version. Vuetify 3 works only with Vue 3; Vuetify 2 (Vue 2) is now end-of-life and receives no updates, not even security fixes[^3]. There is no gradual bridge between the two — the v2→v3 transition was a ground-up rewrite.

Vuetify occupies the "batteries-included, Material" niche in the Vue ecosystem, the rough analogue of MUI in React. Teams reach for it when they want a consistent, prebuilt UI and are comfortable with Material aesthetics; they avoid it when bundle size, a custom design system, or design differentiation is the priority.

## Getting Started

```bash
# Scaffold a new project (installs Vuetify + Vite preset)
npm create vuetify@latest
# or: pnpm create vuetify / yarn create vuetify / bun create vuetify
```

Adding Vuetify to an existing Vue 3 + Vite app:

```ts
// main.ts
import { createApp } from 'vue'
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'
import 'vuetify/styles'
import App from './App.vue'

const vuetify = createVuetify({ components, directives })

createApp(App).use(vuetify).mount('#app')
```

```vue
<!-- App.vue -->
<template>
  <v-app>
    <v-main>
      <v-container>
        <v-btn color="primary" @click="count++">Clicked {{ count }}</v-btn>
      </v-container>
    </v-main>
  </v-app>
</template>

<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>
```

Importing all components as above is fine for prototyping but ships the whole library. For production, use the `vite-plugin-vuetify` auto-import so only referenced components are bundled (see Production Notes).

## Architecture / How It Works

Vuetify is a monorepo. The published package (`vuetify`) is authored in TypeScript and Vue SFCs and compiled to ESM; components live under `vuetify/components` and low-level building blocks (`VApp`, layout, theme) form the framework's runtime spine.

- **`createVuetify()`** returns a Vue plugin that installs global configuration: the active theme, default prop presets, i18n adapter, icon set, and display (breakpoint) service. Components read this configuration through Vue's provide/inject, which is why most Vuetify components must be rendered inside a `<v-app>` root — that component establishes the layout and theme context the rest of the tree depends on.
- **Composition API internals.** Vuetify 3 is built on Vue 3's Composition API. Cross-cutting behavior (theme, display/breakpoints, defaults, form validation, selection groups) is factored into composables that components consume, rather than mixins as in v2.
- **Theme system.** Themes are defined in JS as named palettes; Vuetify computes CSS custom properties (`--v-theme-*`) at runtime, so light/dark switching and per-component theme overrides happen without recompiling CSS.
- **Styling and SASS.** Component styles are authored in SASS. Deep visual customization is done by overriding SASS variables, which requires Vuetify's styles to be compiled *from source* in your build rather than consuming the precompiled CSS — a meaningful build-time cost (see Production Notes).
- **Tree-shaking.** Because importing `vuetify/components` wholesale is large, the intended production path is `vite-plugin-vuetify` (successor to the older `vuetify-loader`[^4]), which rewrites templates to auto-import only the components actually used and wires up on-demand styles.
- **Labs.** Newer or unstable components ship under `vuetify/labs` with an explicit no-stability-guarantee contract; their APIs can change between minor releases before graduating to the main entry point.

The result is that Vuetify's "how it works" is really "how its global configuration and styling pipeline work" — individual components are straightforward Vue, but they assume the `v-app` context, the theme service, and (for a lean bundle) the build-time auto-import plugin.

## Production Notes

**Bundle size is the first thing to get right.** A naive global registration pulls the entire component set. Always use `vite-plugin-vuetify` (or manual per-component imports) so tree-shaking works, and be aware that some features (fonts, Material Design Icons) are separate weight you opt into.

**SASS variable customization slows builds.** The moment you override SASS variables, the build must compile Vuetify's styles from source. On non-trivial apps this noticeably increases cold `dev` startup and build times, and is a common surprise. Prefer the runtime theme system (JS colors + CSS variables) for color changes; reserve SASS-variable overrides for structural style changes that the theme system can't express.

**The v2 → v3 upgrade is a rewrite, not a bump.** Vuetify 3 changed component names, prop names, the grid, slot conventions, and dropped Vue 2. There is no automatic codemod that covers the whole surface; large v2 apps typically migrate screen-by-screen or stay on v2. Since v2 is EOL[^3], staying put means running unmaintained UI code — commercial extended support for v2 is available only through a third-party partner (HeroDevs).

**`v-app` context is mandatory.** Components rendered outside the `v-app` tree (portals, some test setups, embedding Vuetify widgets in a non-Vuetify shell) can mis-render or throw because the theme/layout injection is missing. This bites teams embedding a single Vuetify component into an otherwise non-Vuetify page.

**Data tables at scale.** `v-data-table` is convenient but renders all rows by default; large datasets need `v-data-table-virtual` (virtual scrolling) or server-side pagination (`v-data-table-server`) to stay responsive.

**Labs components can churn.** Anything imported from `vuetify/labs` may change API between minors. Pin versions and read release notes before upgrading if you depend on labs components.

**SSR / Nuxt.** SSR works but needs care around theme and SASS setup; for Nuxt, use the official `vuetify-nuxt-module` rather than wiring the plugin by hand.

## When to Use / When Not

**Use when:**
- You want a complete, consistent UI out of the box and Material Design is acceptable (or desired).
- You're on Vue 3 and value breadth of components (tables, pickers, layout) over minimal footprint.
- Your team would rather configure a theme than build and maintain a component library.
- You need built-in i18n, RTL, breakpoints, and accessibility affordances without assembling them.

**Avoid when:**
- Bundle size is a hard constraint (marketing sites, size-budgeted embeds) — Vuetify is heavy relative to headless or minimal kits.
- You need a distinctive, non-Material design system — retheming Vuetify to *not* look Material is more work than starting from an unstyled library.
- You're still on Vue 2 and can't migrate — Vuetify 2 is EOL; adopting it now is adopting unmaintained code.
- You want to own your markup and styling — a headless component set (behavior without styles) fits that better.

## Alternatives

- `quasarframework/quasar` — Vue 3 framework with a Material-leaning kit plus multi-target builds (SPA/SSR/PWA/Electron/mobile). Use instead when you want one codebase to ship to native/desktop as well as web.
- `primefaces/primevue` — very large Vue 3 component set with swappable themes and an unstyled mode. Use instead when you want design-system flexibility and aren't committed to Material.
- `element-plus/element-plus` — Vue 3 kit with a desktop/admin, non-Material aesthetic. Use instead for internal dashboards and back-office tools.
- `tusen-ai/naive-ui` — Vue 3, TypeScript, tree-shakable, themed in JS with no global CSS reset. Use instead when you want a lighter, non-Material library that's friendly to customization.
- `vuecomponent/ant-design-vue` — Ant Design for Vue. Use instead when you want the Ant enterprise look and its dense data-heavy components.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-03 | First stable release on Vue 2. |
| 2.0 | 2019-07 | Treeshaking, revised theming, major API pass (Vue 2). |
| 3.0 "Titan" | 2022-11 | Full rewrite for Vue 3 on the Composition API; new grid, TypeScript-first, `labs` channel[^5]. |
| 2.x EOL | 2025 | Vuetify 2 declared end-of-life; no further updates or security fixes[^3]. |

## References

[^1]: Vuetify documentation — homepage and component reference. https://vuetifyjs.com
[^2]: Vuetify README, "Supporting Vuetify" (GitHub Sponsors / Open Collective / Tidelift). https://github.com/vuetifyjs/vuetify
[^3]: Vuetify README, "Vuetify 2 Support" — v2 is End Of Life, commercial support via HeroDevs. https://github.com/vuetifyjs/vuetify
[^4]: `vite-plugin-vuetify` — automatic component import and on-demand styles. https://github.com/vuetifyjs/vuetify-loader
[^5]: Vuetify release notes. https://vuetifyjs.com/getting-started/release-notes/

## Tags

vue, vue3, ui-components, component-library, material-design, typescript, frontend, design-system, theming, javascript
