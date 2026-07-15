# element-plus/element-plus

> The Vue 3 successor to Element UI — a broad, enterprise-oriented desktop component library from the Element team.

[GitHub repo](https://github.com/element-plus/element-plus) ·
[Official website](https://element-plus.org) ·
[License: MIT](https://github.com/element-plus/element-plus/blob/dev/LICENSE)

## Overview

Element Plus is a Vue 3 UI component library maintained by the Element team,
originally out of Eleme (饿了么) in China[^1]. It is the successor to Element UI,
the widely deployed Vue 2 library, and is a near-complete rewrite in TypeScript
around Vue 3's Composition API and `<script setup>`. The scope is deliberately
large: some sixty-plus components covering forms, tables, layout, feedback,
navigation, date/time pickers, and a virtualized table/select for large data
sets. The design language is a conventional desktop admin/dashboard aesthetic —
this is a tool for internal tooling, back-office consoles, and CRUD apps, not a
marketing-site or mobile-first kit.

The first production-stable release landed on February 7, 2022[^2], after a long
beta period during which the API churned. Today it is one of the default choices
for Vue 3 component libraries and is heavily used in the Chinese Vue ecosystem,
which shapes its priorities and its documentation. That is the library's defining
tension: it is mature, complete, and API-stable, but its community, issue
tracker, and much of its tribal knowledge are Chinese-first, so English-speaking
teams periodically hit docs gaps and issues resolved only in Chinese threads.

## Getting Started

```bash
npm install element-plus @element-plus/icons-vue
```

```ts
// main.ts — full import (simplest; largest bundle)
import { createApp } from 'vue'
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'
import App from './App.vue'

createApp(App).use(ElementPlus).mount('#app')
```

```vue
<!-- On-demand usage (recommended for production; see below) -->
<template>
  <el-button type="primary" @click="visible = true">Open</el-button>
  <el-dialog v-model="visible" title="Hello">Element Plus</el-dialog>
</template>

<script setup lang="ts">
import { ref } from 'vue'
const visible = ref(false)
</script>
```

## Architecture / How It Works

The repo is a pnpm monorepo. The pieces worth knowing:

- **`packages/components`** — one directory per component, each a self-contained
  Vue SFC + composables + types. Components lean heavily on shared composables in
  **`packages/hooks`** (form context, ids, locale, namespace) and shared
  primitives in **`packages/utils`**.
- **`packages/theme-chalk`** — the styling layer, written in SCSS with BEM class
  naming. As of the 2.x line, tokens are exposed as CSS custom properties
  (`--el-color-primary`, etc.), which is what makes runtime theming and dark mode
  possible. SCSS variables still exist for compile-time overrides.
- **Icons live in a separate package**, `@element-plus/icons-vue`, and are not
  auto-registered — you import and register the ones you use.
- Positioning for tooltips, dropdowns, poppers, and selects is handled by a
  Popper-based floating layer[^3]; form validation delegates to `async-validator`;
  date/time components use `dayjs`.

Theming is dual-track and this trips people up. **CSS-variable theming** works at
runtime (toggle a class, swap `--el-*` values) with no build step. **SCSS-variable
theming** overrides the source tokens, requires your bundler to compile Element
Plus's SCSS with your override map injected, cannot happen at runtime, and
interacts with the on-demand import setup below.

## Production Notes

**On-demand import is effectively mandatory.** A full `.use(ElementPlus)` import
pulls the entire library and its CSS into your bundle. For real apps the
recommended path is auto-import via `unplugin-vue-components` and
`unplugin-auto-import` with the `ElementPlusResolver`, which registers only the
components you reference and pulls their styles per-component[^4]. Getting SCSS
theme overrides to apply through this resolver requires the resolver's
`importStyle: 'sass'` option plus an `additionalData` SCSS entry — a
frequently-hit configuration snag.

**SSR / Nuxt needs the official module.** Use `@element-plus/nuxt`[^5] rather than
wiring the plugin by hand; without it you get style-ordering and hydration
mismatches, and components touching `window`/`document` need care.

**Locale and global config live in `ElConfigProvider`.** Non-English apps must
pass a locale there — date pickers, pagination, and empty states pull strings
from it. It is also where you set global component `size` and the base `z-index`,
worth setting once rather than per-component.

**Bundle and CSS weight.** Even with tree-shaking, the CSS for heavily-styled
components (table, date-picker, select) is substantial. The virtualized
`el-table-v2` exists precisely because the standard `el-table` degrades on large
row counts; reach for it before hand-rolling pagination hacks.

**Version drift.** The 2.x line is API-stable, but minor releases occasionally
ship behavior changes or prop deprecations, and the default branch is `dev`, so
pin an exact version and read release notes before bumping. Node >= 20 is
required by recent releases[^6].

## When to Use / When Not

**Use when:**
- You're building a Vue 3 admin panel, internal tool, or data-heavy CRUD app and
  want a complete component set without assembling one.
- You (or your team) came from Element UI and want the closest Vue 3 continuation.
- You need mature tables, forms with validation, and date pickers out of the box.

**Avoid when:**
- You want a distinctive or brand-forward visual design — Element Plus looks like
  an admin console, and deep restyling fights the SCSS token system.
- You're mobile-first or building a marketing site; this is a desktop kit.
- You need a fully headless/unstyled primitive layer for a custom design system.
- English-only support with fast English issue turnaround is a hard requirement.

## Alternatives

- vuetify/vuetify — use instead when you want Material Design and a larger,
  more opinionated visual system.
- vueComponent/ant-design-vue — use when your team standardizes on Ant Design
  and wants the enterprise-console look with parity to the React version.
- primefaces/primevue — use when you need an even broader component catalog with
  swappable themes and an unstyled mode.
- tusen-ai/naive-ui — use when you want a TypeScript-first Vue 3 library with
  JS-object theming instead of SCSS.
- unovue/reka-ui — use when you want headless, unstyled primitives to build your
  own design system rather than a pre-styled kit.

## History

| Version | Date | Notes |
|---------|------|-------|
| Element UI 1.0 | 2016 | Vue 2 predecessor from the Eleme team[^1]. |
| 1.0.0-beta | 2020–2021 | Vue 3 + TypeScript rewrite; long, churning beta. |
| 2.0.0 | 2026-02-07 (2022) | First production-stable release; API frozen[^2]. |
| 2.x | ongoing | CSS-variable theming, dark mode, `el-table-v2`, Nuxt module. |

## References

[^1]: Element project background — Element team (originally Eleme). https://element-plus.org/en-US/guide/design.html
[^2]: Element Plus README, "Breaking Change List" — first stable release February 7, 2022. https://github.com/element-plus/element-plus#breaking-change-list
[^3]: Element Plus source, `packages/components` (popper/tooltip/select positioning layer). https://github.com/element-plus/element-plus/tree/dev/packages/components
[^4]: Element Plus docs, "Quick Start — On-demand Import." https://element-plus.org/en-US/guide/quickstart.html
[^5]: `@element-plus/nuxt` module. https://github.com/element-plus/element-plus-nuxt
[^6]: Element Plus README node engine badge (node >= 20). https://github.com/element-plus/element-plus

## Tags

vue, vue3, typescript, ui-library, component-library, design-system, frontend, scss, admin-dashboard, element-ui
