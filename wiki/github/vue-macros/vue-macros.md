# vue-macros/vue-macros

> A collection of experimental macros and syntax sugar for Vue — the proving ground where features graduate into Vue core.

[GitHub repo](https://github.com/vue-macros/vue-macros) ·
[Official website](https://vue-macros.dev) ·
[License: MIT](https://github.com/vue-macros/vue-macros/blob/main/LICENSE)

## Overview

Vue Macros is a build-time plugin that adds compiler macros and syntax sugar to Vue single-file components (SFCs) and JSX beyond what Vue core ships. It is authored primarily by Kevin Deng (`sxzz`), a Vue core team member, which explains its defining role: it is the sandbox where new SFC ergonomics are prototyped in the wild before some of them are absorbed into Vue itself[^1]. The repository moved from `sxzz/unplugin-vue-macros` to the `vue-macros` organization; the old path still redirects.

The list of graduated features is the clearest way to understand the project. `defineOptions` originated here and landed in Vue 3.3[^2]. `defineModel` was prototyped here and became stable in Vue 3.4[^3]. Reactivity Transform (`$ref`, `$computed`) went the other direction — it was an experimental Vue core feature, was removed from core in 3.4, and its maintenance moved into Vue Macros[^3]. This churn is the central tradeoff: adopting a Vue Macros feature is adopting something that may become a core built-in (making the dependency redundant), may change shape, or may be deprecated. It is deliberately ahead of the stability curve.

Everything is delivered through [unjs/unplugin](https://github.com/unjs/unplugin), so the same macros work across Vite, Nuxt, webpack, Rspack, Rollup, esbuild, Vue CLI, and rsbuild. Both Vue 2.7 and Vue 3 are supported.

## Getting Started

```bash
npm i -D unplugin-vue-macros
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import Vue from '@vitejs/plugin-vue'
import VueMacros from 'unplugin-vue-macros/vite'

export default defineConfig({
  plugins: [
    VueMacros({
      plugins: {
        vue: Vue(),          // Vue Macros wraps the official Vue plugin
      },
    }),
  ],
})
```

```vue
<script setup lang="ts">
// defineOptions: set component options inside <script setup>
defineOptions({ name: 'MyComponent', inheritAttrs: false })

// defineModel: two-way binding without manual props + emits wiring
const count = defineModel<number>()
</script>

<template>
  <button @click="count!++">{{ count }}</button>
</template>
```

Note: `defineOptions` and `defineModel` above now exist in Vue core; they are shown because they are the canonical examples of what Vue Macros pioneered. The differentiated value today is in the not-yet-graduated macros.

## Architecture / How It Works

Vue Macros is a monorepo of small, single-purpose packages (one macro per package, e.g. `@vue-macros/define-options`, `@vue-macros/short-vmodel`, `@vue-macros/reactivity-transform`, `@vue-macros/jsx-directive`). The `unplugin-vue-macros` / `vue-macros` meta-package aggregates them and wires them into unplugin.

Each macro is a source transform. Vue Macros intercepts SFC `<script setup>` blocks (and, for some macros, JSX/TSX) and rewrites them into plain Vue-compatible code before Vue's own compiler runs. Because it wraps `@vitejs/plugin-vue` rather than sitting beside it, ordering is explicit — you pass the Vue plugin *into* Vue Macros. It reaches into Vue's compiler internals (`@vue/compiler-sfc`), which is why Vue Macros versions are coupled to Vue versions.

There is a second, easy-to-miss half: **editor/type support**. Because macros generate code that does not exist in the source the TypeScript server sees, IDE type-checking and `vue-tsc` need a separate Volar plugin (`@vue-macros/volar`) registered in `tsconfig.json`. Without it, valid macro usage shows as type errors in the editor even though the build succeeds. This two-piece setup (build plugin + Volar plugin) is the most common source of "it compiles but my editor is red" confusion.

Most macros are opt-in and disabled by default; you enable them individually through the plugin's `macros` / feature options.

## Production Notes

- **Volar plugin drift.** The build transform and the `@vue-macros/volar` type plugin are versioned separately in practice and must stay aligned with each other and with your Volar/`vue-tsc` version. Mismatches produce editor-only type errors that do not fail the build, or vice versa. Budget setup time for the type side, not just the Vite side.
- **Coupling to Vue internals.** Vue Macros depends on `@vue/compiler-sfc` internals. Upgrading Vue (especially minor versions that touch the SFC compiler) can require a matching Vue Macros bump; pin both and upgrade them together.
- **Feature redundancy after graduation.** If you adopted `defineOptions` or `defineModel` via Vue Macros before Vue 3.3/3.4, those are now core. Continuing to route them through Vue Macros adds a transform and a dependency for no benefit — audit and drop graduated macros on upgrade.
- **Reactivity Transform is legacy.** `$ref`/`$computed`-style Reactivity Transform was rejected by Vue core and is discouraged even inside Vue Macros. Do not start new code on it; it exists mainly to keep existing users working[^3].
- **Plugin ordering.** Because Vue Macros wraps the Vue plugin, other plugins that also expect to run before/after `@vitejs/plugin-vue` (component auto-import, i18n, etc.) can end up in the wrong position. Order problems surface as macros silently not transforming.
- **Nuxt.** A Nuxt module (`unplugin-vue-macros/nuxt`) exists; enable macros through Nuxt config rather than hand-wiring unplugin.

## When to Use / When Not

**Use when:**
- You want SFC ergonomics that Vue core does not yet ship (e.g. `defineProp`/`defineProps` refinements, `shortVmodel`, `setupSFC`, JSX directives, `defineRender`, `defineStyleX`).
- You are on a fast-moving internal codebase and can absorb macro churn on Vue upgrades.
- You want a single unplugin-based install that works identically across Vite, webpack, Rspack, and Rollup.

**Avoid when:**
- You only need `defineModel` / `defineOptions` — use Vue 3.4+ core and skip the dependency entirely.
- You need long-term API stability with rare breaking changes; this project is intentionally experimental.
- You cannot invest in the Volar/type-plugin setup and would ship editor-red code.
- Your team dislikes non-standard SFC syntax that new contributors won't recognize.

## Alternatives

- vuejs/core — once a macro graduates (`defineModel`, `defineOptions`), use the core built-in and drop Vue Macros for that feature.
- vue-vine/vue-vine — use instead when you want a fundamentally different Vue component authoring model, not incremental macros on top of SFCs.
- unplugin/unplugin-auto-import — use instead when all you want is auto-importing Vue/Composition APIs, not new SFC syntax.
- vuejs/babel-plugin-jsx — use instead when you only need Vue JSX transforms and none of the macro layer.
- sxzz/unplugin-vue-components — use alongside/instead when the goal is on-demand component registration rather than script-setup macros.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2022-02 | First release as `sxzz/unplugin-vue-macros`; early `defineOptions`, Reactivity Transform experiments[^1]. |
| — | 2023-05 | `defineOptions` adopted into Vue 3.3 core[^2]. |
| — | 2023-12 | `defineModel` stabilized in Vue 3.4 core; Reactivity Transform removed from core and maintained here[^3]. |
| repo move | — | Migrated to the `vue-macros` GitHub org; monorepo of per-feature packages under `@vue-macros/*`. |

(Vue Macros publishes many independently-versioned packages; there is no single project-wide semver line, so this table tracks ecosystem milestones rather than a single version number.)

## References

[^1]: Vue Macros documentation and repository. https://vue-macros.dev
[^2]: Vue.js blog, "Announcing Vue 3.3" (introduces `defineOptions` in `<script setup>`). https://blog.vuejs.org/posts/vue-3-3
[^3]: Vue.js blog, "Announcing Vue 3.4" (stabilizes `defineModel`, removes Reactivity Transform). https://blog.vuejs.org/posts/vue-3-4

## Tags

vue, vuejs, macros, compiler-macros, script-setup, unplugin, vite, sfc, typescript, syntax-sugar, build-tooling
