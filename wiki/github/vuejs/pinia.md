# vuejs/pinia

> The officially recommended state store for Vue 3 — a thin, type-safe layer over the Composition API reactivity system that replaced Vuex.

[GitHub repo](https://github.com/vuejs/pinia) ·
[Official website](https://pinia.vuejs.org) ·
[License: MIT](https://github.com/vuejs/pinia/blob/v4/LICENSE)

## Overview

Pinia is a state management library for Vue, written by Eduardo San Martin Morote (posva), who also maintains Vue Router. It began in late 2019 as an experiment in what a Vue store would look like if designed around the (then new) Composition API instead of Vuex's options-object-and-mutations model[^1]. It has since become the state solution the Vue team recommends by default, and Vuex is in maintenance mode with its own docs pointing users to Pinia[^2].

The defining design choice is that Pinia is barely a library. A store is a `reactive` object living in a Vue `effectScope`, registered on a root instance that you install with `app.use(pinia)`. There are no mutations, no dispatch strings, and no nested-module namespacing — you read state directly and change it directly or through actions. This makes stores feel like plain composables and gives near-perfect TypeScript inference, which was the practical failure of Vuex. The tradeoff is that Pinia gives you very little structure: it does not enforce where mutations happen, does not manage async/server state, and leaves conventions (folder layout, when a store is warranted) entirely to the team.

Because it is a thin wrapper over Vue's own reactivity, Pinia's behavior — and most of its footguns — are really Vue reactivity behavior surfacing through a store shape. Understanding `ref`/`reactive`/`storeToRefs` matters more here than learning any Pinia-specific API.

## Getting Started

```bash
npm install pinia
```

```js
// main.js — install the root store once per app
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

createApp(App).use(createPinia()).mount('#app')
```

```ts
// stores/counter.ts — Setup store (Composition API style)
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)                       // becomes state
  const double = computed(() => count.value * 2) // becomes a getter
  function increment() { count.value++ }     // becomes an action
  return { count, double, increment }
})
```

```vue
<script setup>
import { storeToRefs } from 'pinia'
import { useCounterStore } from '@/stores/counter'

const store = useCounterStore()
const { count, double } = storeToRefs(store) // keep reactivity when destructuring
const { increment } = store                   // actions can be destructured directly
</script>
```

## Architecture / How It Works

A store can be declared two ways. **Options stores** pass a `{ state, getters, actions }` object, mirroring Vuex's shape. **Setup stores** pass a function that returns `ref`s (state), `computed`s (getters), and functions (actions) — identical to writing a composable. Both compile down to the same reactive object; the setup form gives more flexibility (arbitrary composables inside a store) at the cost of manual return lists.

The root `pinia` instance holds a single reactive `pinia.state` object keyed by store id. Every store you define registers its state under its id in that one tree. This centralization is what makes SSR serialization work: the server renders, dumps `pinia.state.value` to the HTML, and the client rehydrates it before mounting[^3]. Each store also runs inside its own `effectScope`, so its `computed` getters and watchers can be disposed together when the store is torn down.

Instead of mutations, Pinia exposes direct assignment plus three primitives: `$patch` (batch multiple state changes into one reactivity flush and one devtools entry), `$subscribe` (react to state changes), and `$onAction` (hook before/after/on-error of any action). Plugins registered with `pinia.use(plugin)` receive every store on creation and can inject shared properties or wire cross-cutting behavior (persistence, logging) — this is how the persistence and undo/redo ecosystem plugins work. DevTools integration is provided through `@vue/devtools-api`, and hot-module-replacement is opt-in via `acceptHMRUpdate`.

The important mental model: `useStore()` is a function, and it must be called *after* a pinia instance is active (i.e. inside a component `setup`, or after `app.use(pinia)`). It resolves the active pinia via Vue's injection context. Calling it too early, or outside a component in SSR without passing the instance explicitly, is the source of the most common runtime error.

## Production Notes

**`getActivePinia() was called but there was no active Pinia`.** The signature error. It fires when a store is used at module top-level, before `app.use(pinia)`, or in an SSR request handler without the pinia instance in scope. Rule of thumb: never call `useStore()` at the top level of a module — only inside `setup`, an action, or a function that runs after mounting[^4].

**Destructuring silently breaks reactivity.** `const { count } = useStore()` copies the current value and loses the ref binding. State and getters must go through `storeToRefs`; only actions (plain functions) are safe to destructure raw. This is Vue reactivity, not a Pinia bug, but it catches nearly every newcomer.

**SSR cross-request state leakage.** On the server you must create a fresh `createPinia()` per request. Sharing one singleton across requests leaks one user's state into another's — a real data-exposure bug, not just a glitch. The `@pinia/nuxt` module handles this correctly; hand-rolled SSR setups do not by default[^3].

**Getters that use `this` need explicit return types.** In options stores, a getter that reads another getter via `this` will trip TypeScript's circular inference and must be annotated (`doublePlusOne(): number { ... }`). Setup stores sidestep this by using `computed` directly.

**Pinia is not an async/server-state manager.** It has no caching, deduplication, retries, or staleness model. Teams routinely overload stores with fetched data and reinvent a worse TanStack Query. Keep server state in a query library and use Pinia for genuine client/UI state.

**Vue 2 is gone in v3+.** Pinia 2 supported both Vue 2 and Vue 3 through `@vue/composition-api`; the 3.x line dropped Vue 2 entirely, so Vue 2 apps must pin the 2.x line (the `v2` branch)[^5]. The `v4` branch is the current development default.

## When to Use / When Not

**Use when:**
- You are on Vue 3 and need shared state across components with strong TypeScript inference.
- You want stores that read like composables, with devtools timeline and plugin hooks.
- You are doing SSR/Nuxt and need serializable, per-request-isolated global state.
- You are migrating off Vuex and want the officially blessed path.

**Avoid when:**
- Your "state" is really server data — reach for a query/cache library instead.
- The app is small enough that a shared `reactive()` in a composable file suffices; a store library is overhead.
- You need enforced mutation discipline or formal state machines — Pinia deliberately imposes neither.
- You are still on Vue 2 and can't move to the frozen 2.x line.

## Alternatives

- vuejs/vuex — the predecessor, now in maintenance mode; use only to maintain an existing Vuex codebase or a Vue 2 app.
- vueuse/vueuse — `createGlobalState` and shared composables cover tiny global state without adding a store abstraction.
- tanstack/query — use for server/async state (caching, refetch, dedup) instead of stuffing fetched data into a Pinia store.
- statelyai/xstate — use when the logic is genuinely a state machine and you want explicit, enforced transitions rather than freeform mutable state.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-11-18 | Repo created as a Composition API store experiment[^1]. |
| 2.0 | 2021-11 | First stable major; dual Vue 2 / Vue 3 support; becomes the recommended Vue store[^2]. |
| 3.0 | 2025 | Drops Vue 2 support; requires Vue 3 and modern tooling[^5]. |
| 4.x | current | Active development line (`v4` default branch, npm `^4`). |

## References

[^1]: Pinia introduction & motivation. https://pinia.vuejs.org/introduction.html
[^2]: Vue.js docs, "State Management" — Pinia recommended over Vuex. https://vuejs.org/guide/scaling-up/state-management.html
[^3]: Pinia docs, "Server Side Rendering (SSR)". https://pinia.vuejs.org/ssr/
[^4]: Pinia docs, "Defining a Store" — usage constraints of `useStore()`. https://pinia.vuejs.org/core-concepts/
[^5]: Pinia migration / v2 branch for Vue 2. https://github.com/vuejs/pinia/tree/v2

## Tags

vue, state-management, store, typescript, composition-api, ssr, nuxt, frontend, vuex-alternative, reactivity
