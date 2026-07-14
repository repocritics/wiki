# quasarframework/quasar

> A Vue 3 component library and build system that targets SPA, SSR, PWA, mobile, desktop, and browser extensions from one codebase.

[GitHub repo](https://github.com/quasarframework/quasar) ·
[Official website](https://quasar.dev) ·
[License: MIT](https://github.com/quasarframework/quasar/blob/dev/LICENSE)

## Overview

Quasar is two things bundled under one name: a large Material-Design-based Vue 3 component library (components prefixed `Q`, e.g. `QBtn`, `QTable`, `QDialog`), and an opinionated build system that compiles a single source tree into six deployment targets — SPA, server-side-rendered app, PWA, Cordova/Capacitor mobile app, Electron desktop app, and browser extension (BEX)[^1]. Created and still led by Razvan Stoenescu, it has been in development since 2015[^2] and is MIT-licensed.

The project's defining bet is the "write once, deploy everywhere" premise: the same `.vue` components, the same router, and the same store drive every target, with per-target configuration living in a single `quasar.config` file. This is Quasar's biggest draw for small teams — one skillset covers web, mobile, and desktop — and its biggest source of coupling. You are not adopting a component library; you are adopting Quasar's CLI, its config format, its build modes, and its opinions about how a project is laid out.

With roughly 27k stars and near-daily commits to the `dev` branch as of 2026, Quasar is actively maintained but sits in the second tier of Vue tooling popularity behind Nuxt and Vuetify. Its component catalog is unusually broad — data tables, virtual scroll, date/time pickers, notifications, and a full layout system are in-box rather than plugins — which is the practical reason teams pick it over assembling equivalents.

## Getting Started

```bash
npm i -g @quasar/cli
npm create quasar@latest    # scaffolds a project; prompts for Vite vs Webpack, TS, etc.
cd my-app
quasar dev                  # runs the SPA target by default
```

```vue
<!-- src/pages/IndexPage.vue -->
<template>
  <q-page padding>
    <q-btn color="primary" label="Notify" @click="ping" />
    <q-table :rows="rows" :columns="columns" row-key="id" />
  </q-page>
</template>

<script setup>
import { useQuasar } from 'quasar'
const $q = useQuasar()
const rows = [{ id: 1, name: 'Tom' }, { id: 2, name: 'Brad' }]
const columns = [{ name: 'name', label: 'Name', field: 'name' }]
function ping () {
  $q.notify({ message: 'Saved', color: 'positive' })
}
</script>
```

Building for another target is a mode switch rather than a rewrite:

```bash
quasar build              # SPA
quasar build -m ssr       # server-side rendered
quasar build -m pwa       # PWA
quasar build -m electron  # desktop
quasar build -m capacitor # mobile
```

## Architecture / How It Works

Quasar is layered into a runtime library and a build layer, and the two are versioned and shipped separately.

- **`quasar`** — the runtime npm package: components, directives, the `$q` plugin object (Notify, Dialog, Loading, Dark, screen breakpoints), and a CSS layer (a flexbox grid, spacing utilities, typography, and a theming system built on CSS custom properties). This package can be consumed standalone in a plain Vite + Vue project via `@quasar/vite-plugin`, without adopting the CLI[^1].
- **`@quasar/app-vite` / `@quasar/app-webpack`** — the build system. Each wraps the underlying bundler (Vite or Webpack) and injects Quasar's boot files, the multi-target build modes, and auto-import of components/directives so you do not register them by hand. `@quasar/app-vite` is the current default; the Webpack variant remains for legacy projects.

Cross-target support is achieved by swapping entry points and adapters per mode while keeping `src/` shared. SSR runs an Express-style Node server that Quasar generates; PWA mode layers Workbox on top of the SPA output; Electron and Capacitor modes wrap the built web assets with their respective shells and expose native APIs through Quasar's boot/plugin conventions. "Boot files" are Quasar's initialization hook — ordered modules that run before the root Vue app mounts, used for installing plugins, axios instances, i18n, and auth guards.

The theming system is compile-time and run-time: a Sass variables layer sets brand colors and breakpoints at build time, while `$q.dark` and CSS variables allow runtime dark-mode and dynamic theming. Icon and font sets ship separately in `@quasar/extras`, and `@quasar/icongenie` generates app icons and splash screens for the mobile/desktop targets.

The tight point is that the CLI, the config file, and the component library co-evolve. Upgrading the framework often means upgrading `@quasar/app-*` in lockstep, because build-mode behavior and component APIs are released together.

## Production Notes

**Bundle size and tree-shaking.** The component set is large, and while `@quasar/app-*` auto-imports only what you use, apps that lean on many components ship non-trivial JS/CSS. Audit the built output; the biggest wins come from limiting icon-set imports (importing a whole icon font is a common footgun) and from not pulling in components you only use once.

**SSR is the sharp edge.** SPA and PWA modes are close to turnkey; SSR is where the "one codebase" promise leaks. Server-only vs client-only code, hydration mismatches, and boot files that assume `window` all surface here. Third-party libraries that touch the DOM at import time need client-only guarding. Budget real time for the first SSR deploy.

**Mobile/desktop targets inherit their shells' problems.** Capacitor and Electron modes mean you also own Capacitor's and Electron's release, signing, and native-plugin realities. Quasar smooths the build wiring, not the platform-specific store submission, native permissions, or auto-update concerns.

**Config is centralized and stateful.** `quasar.config` controls framework plugins, boot files, build options, and per-mode settings. It is powerful but a single point of coupling; changing bundlers or modes is a config surgery, not a drop-in.

**Version pinning across the toolchain.** Because `quasar`, `@quasar/app-vite`, `@quasar/extras`, and `@quasar/cli` release on independent cadences, mismatched versions are a recurring source of confusing build errors. Keep them upgraded together and read the upgrade guide between minors.

**Ecosystem depth.** Quasar's community and plugin ecosystem is smaller than Nuxt's or Vuetify's. For niche needs you are more likely to be first, which means fewer copy-paste answers and more reading of source.

## When to Use / When Not

**Use when:**
- One small team needs to ship web, mobile, and desktop from a shared Vue codebase.
- You want a broad, batteries-included component set (tables, pickers, layout, notifications) without assembling it from separate libraries.
- You are comfortable adopting a full CLI + config system, not just a component dependency.
- You want a Material-Design-flavored UI without going all-in on Nuxt's rendering opinions.

**Avoid when:**
- You only need components in an existing build — use `@quasar/vite-plugin` for the library alone, or a lighter component set.
- Your priority is SSR/edge-first content with a large plugin ecosystem — Nuxt is the deeper meta-framework there.
- You need a design system other than Material without heavy overriding.
- You want a large community with abundant tutorials for every edge case.

## Alternatives

- nuxt/nuxt — the mainstream Vue meta-framework; deeper SSR/edge and module ecosystem, but no built-in cross-platform mobile/desktop targets. Use when web (SSR/SSG) is the whole story.
- vuetifyjs/vuetify — the most popular Material Design Vue component library; use when you want components only and not a build system or multi-target CLI.
- primefaces/primevue — large themeable Vue component suite; use when you want breadth of components with design-agnostic theming.
- ionic-team/ionic-framework — cross-platform (Vue/React/Angular) with a strong mobile/Capacitor story; use when mobile is the primary target and web is secondary.
- element-plus/element-plus — desktop-web-oriented Vue 3 component library; use for admin/dashboard UIs where mobile and native are not requirements.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015–2018 | Early releases; component library atop Vue 2. |
| 1.0 | 2019-07 | First major release; Vue 2, unified CLI build modes[^2]. |
| 2.0 | 2021-06 | Vue 3 rewrite; Composition API support, `$q` plugin model[^3]. |
| — | 2021–2022 | `@quasar/app-vite` introduced alongside the Webpack build system. |

## References

[^1]: Quasar documentation — introduction, build modes, and Vite plugin. https://quasar.dev/
[^2]: quasarframework/quasar repository, created 2015-10-05; maintained by Razvan Stoenescu. https://github.com/quasarframework/quasar
[^3]: Quasar v2 (Vue 3) upgrade guide. https://quasar.dev/start/upgrade-guide

## Tags

vue, vue3, component-library, material-design, cross-platform, ssr, pwa, electron, capacitor, javascript, frontend-framework, ui-toolkit
