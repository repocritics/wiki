# vuejs/core

> The progressive JavaScript framework — Vue 3 reactivity core and renderer.

[GitHub repo](https://github.com/vuejs/core) ·
[Official website](https://vuejs.org) ·
[License: MIT](https://github.com/vuejs/core/blob/main/LICENSE)

## Overview

`vuejs/core` is the Vue 3 monorepo containing `@vue/reactivity`, `@vue/runtime-core`, `@vue/runtime-dom`, the SFC compiler, and the server renderer. It was a near-total rewrite of Vue 2, released in 2020 with a TypeScript-first codebase and a Proxy-based reactivity system[^1]. The repo separates the reactivity package from the renderer, which is unusual in this class of libraries — `@vue/reactivity` is consumable on its own and is used by other projects (Solid's `solid-js/store`, Pinia, and several signal experiments draw from it).

Vue's distinctive surface is the **single-file component** (`.vue`) combining template, script, and scoped style in one file, and the **Composition API** with `<script setup>` syntax that became the recommended default in 2022[^2]. The Options API still works and is not deprecated, but new docs lead with composition. Vue's reactivity is signal-flavored (it tracks dependencies through effect functions) without using the "signals" name.

Vapor Mode — a no-virtual-DOM compilation target that emits direct DOM mutation code, conceptually similar to Svelte 5 and Solid — has been in active development through 2025 but is not yet stable[^3].

## Getting Started

```bash
npm create vue@latest my-app
cd my-app
npm install
npm run dev
```

A minimal Composition API component:

```vue
<script setup>
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<template>
  <button @click="count++">
    count: {{ count }} (doubled: {{ doubled }})
  </button>
</template>
```

## Architecture / How It Works

Vue 3's reactivity is built on ES2015 `Proxy`. When you call `reactive({})` or `ref(0)`, Vue wraps the value in a proxy that intercepts reads (`get`) and writes (`set`/`deleteProperty`). Reads inside a tracking scope register a dependency; writes trigger any registered effects[^4]. This is fundamentally the same model as MobX, Solid signals, and Svelte 5 runes — the differences are in ergonomics (`.value` unwrapping vs. function calls vs. compiler macros) and granularity.

The renderer is virtual-DOM-based, but the SFC compiler performs aggressive static analysis to mark nodes as static (hoisted out of render functions) or to patch only specific dynamic bindings (`PatchFlags`). The net effect is that updates do diffing only on the parts of a template the compiler couldn't prove static.

`<script setup>` is a compile-time transform: refs declared in the setup script become template bindings without `return`, `defineProps` and `defineEmits` are macros recognized by the compiler, and the whole block runs once per instance. This is the same macro-as-compile-time-rune pattern Svelte 5 uses.

Pinia replaced Vuex as the official state management library in 2022[^5]. It uses the same Composition API primitives and has no notion of mutations/actions split — stores are just composables.

## Production Notes

**Vue 2 → 3 migration.** The breaking changes are extensive: new `app.createApp()` mount, removed `Vue.set`/`Vue.delete` (Proxy reactivity makes them unnecessary), filter syntax removed, `v-model` semantics changed, async components and functional components rewritten. Vue 2.7 ("Naruto") backported the Composition API as a migration bridge[^6]. End-of-life for Vue 2 was 2023-12-31; LTS extended support is paid via Herodevs.

**Reactivity gotchas.** Destructuring a reactive object loses reactivity (`const { x } = reactive({x:1})` gives a plain number). Use `toRefs` / `storeToRefs` for Pinia stores. Mutating arrays at an index works in Vue 3 (it didn't in Vue 2 without `Vue.set`).

**Bundle size.** Tree-shakeable, with `defineCustomElement` and lazy-loaded i18n. A non-trivial Vue 3 app lands around 30–60 KB gzipped runtime, somewhat smaller than React + react-dom but larger than Svelte or Solid.

**Ecosystem.** Vue's ecosystem is largest in Chinese-speaking markets; Element Plus, Naive UI, Ant Design Vue, and Arco Design have richer component sets than most Western Vue libraries. Western ecosystem is dominated by Nuxt (framework), VueUse (composables), and shadcn-vue (port).

**Nuxt coupling.** Nuxt is the de facto Vue meta-framework, similar to Next.js for React. Nuxt 3 is built on Nitro (a deployment-agnostic server) and ships hybrid rendering, server routes, and module ecosystem. Production Vue work outside Nuxt is rarer than production React outside Next.

## When to Use / When Not

**Use when:**
- You want SFCs with template syntax rather than JSX.
- Your audience or hiring pool is heavy in APAC markets.
- You need a reactivity primitive consumable outside the framework (`@vue/reactivity`).
- You're building admin tools or content sites where Nuxt's hybrid rendering fits.

**Avoid when:**
- You need React Native or a serious cross-platform native story (NativeScript-Vue and Quasar Mobile exist but lag).
- You need first-class signals exposed in the application API (Vue's signals are internal — `ref`/`reactive` are the user-facing wrapper).
- You're size-constrained below ~30 KB runtime.

## Alternatives

- [facebook/react](../facebook/react.md) — larger ecosystem, JSX instead of templates, heavier runtime.
- [sveltejs/svelte](../sveltejs/svelte.md) — compile-away runtime, smaller bundles, smaller ecosystem.
- [solidjs/solid](../solidjs/solid.md) — explicit signals + JSX, no virtual DOM.
- [withastro/astro](../withastro/astro.md) — can host Vue islands without committing the whole app.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.6 | 2013-12 | First Evan You release. |
| 1.0 | 2015-10 | "Evangelion" — production-ready. |
| 2.0 | 2016-09 | Virtual DOM, server rendering. |
| 2.6 | 2019-02 | Improved slot syntax, async error handling. |
| 3.0 | 2020-09 | "One Piece" — Proxy reactivity, Composition API, TS rewrite[^1]. |
| 3.2 | 2021-08 | `<script setup>` stable[^2]. |
| 3.3 | 2023-05 | Generic components, defineModel macro. |
| 3.4 | 2024-01 | Parser rewrite, faster reactivity. |
| 3.5 | 2024-09 | Reactive props destructure, deferred teleport. |
| Vapor | 2025+ | No-virtual-DOM compilation target, in development[^3]. |

## References

[^1]: Evan You, "Vue 3.0.0 'One Piece' Released" — 2020-09-18. https://blog.vuejs.org/posts/vue-3-one-piece
[^2]: Vue docs, "`<script setup>`". https://vuejs.org/api/sfc-script-setup.html
[^3]: Evan You, "Vue Vapor: A no virtual DOM mode for Vue" — 2024. https://github.com/vuejs/core-vapor
[^4]: Vue docs, "Reactivity Fundamentals". https://vuejs.org/guide/essentials/reactivity-fundamentals.html
[^5]: Eduardo San Martin Morote, "Pinia officially the new default" — 2022. https://pinia.vuejs.org/introduction.html
[^6]: Vue docs, "Vue 2.7 Naruto". https://blog.vuejs.org/posts/vue-2-7-naruto
