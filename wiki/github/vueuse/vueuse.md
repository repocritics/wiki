# vueuse/vueuse

> A large collection of tree-shakeable Vue 3 composition utilities — the de facto standard library for the Composition API.

[GitHub repo](https://github.com/vueuse/vueuse) ·
[Official website](https://vueuse.org) ·
[License: MIT](https://github.com/vueuse/vueuse/blob/main/LICENSE)

## Overview

VueUse is a collection of several hundred reactive utility functions ("composables") for Vue 3, created by Anthony Fu in December 2019 and openly inspired by React's `streamich/react-use`[^1]. Each function follows the `useXxx` convention, returns Vue refs, and wires up its own lifecycle cleanup, so a call like `useMouse()` gives you reactive `x`/`y` that update on `mousemove` and detach automatically when the component unmounts. It is the utility layer most Vue 3 apps reach for instead of hand-rolling event listeners, storage sync, media queries, or timers.

The defining tension is breadth versus curation. VueUse is not a focused library with one job; it is a grab-bag that spans DOM events, sensors, browser APIs, animation, state persistence, and framework integrations. Tree-shaking means you only pay bundle cost for what you import, so the sheer count is not a runtime tax — but the API surface is enormous, quality varies across the long tail of functions, and "there's a `useX` for that" can pull you into VueUse for things a three-line composable or a plain utility (lodash, `es-toolkit`) would cover without the dependency.

The second recurring theme is SSR. Many composables touch browser-only globals (`window`, `navigator`, `document`), so they are written to be SSR-safe by deferring access until mount — but that safety is a contract you have to respect, and reaching for `.value` too early during server render is the most common category of VueUse bug reports.

## Getting Started

```bash
npm i @vueuse/core
```

```ts
import { useLocalStorage, useMouse, usePreferredDark } from '@vueuse/core'

const { x, y } = useMouse()          // reactive pointer position
const isDark = usePreferredDark()     // tracks prefers-color-scheme

// reactive state mirrored to localStorage (survives reloads)
const store = useLocalStorage('my-storage', { name: 'Apple', color: 'red' })
```

No bundler is required — `@vueuse/core` and `@vueuse/shared` are also exposed as UMD globals via CDN (`window.VueUse`). A Nuxt module (`@vueuse/nuxt`) auto-imports every composable.

## Architecture / How It Works

VueUse is a monorepo published as several coordinated packages:

- **`@vueuse/shared`** — framework-primitive helpers (`useToggle`, `until`, `tryOnScopeDispose`, ref utilities) shared by everything else.
- **`@vueuse/core`** — the main body: DOM, sensors, browser, state, animation, and network composables. This is the package almost everyone installs.
- **`@vueuse/components`** — renderless component wrappers (`<UseMouse>`, `<OnClickOutside>`) for template-only usage.
- **Add-ons** — opt-in packages that carry heavier peer dependencies: `@vueuse/router`, `@vueuse/integrations` (axios, cookies, focus-trap, jwt, qrcode, etc.), `@vueuse/rxjs`, `@vueuse/firebase`, `@vueuse/electron`, `@vueuse/math`.

Two conventions do most of the structural work. First, functions accept `MaybeRefOrGetter` arguments — a raw value, a ref, or a getter — and normalize them with `toValue`, so a composable is agnostic about whether its input is reactive. Second, cleanup is bound to Vue's effect scope: composables register teardown via `tryOnScopeDispose`/`onScopeDispose` rather than assuming a component instance, which is why they compose inside `effectScope()` and other composables without leaking listeners.

Cross-cutting options recur across the library: **event filters** (`throttleFilter`, `debounceFilter`, `pausableFilter`) let you rate-limit any reactive source, and **configurable targets** let DOM composables attach to `window`, `document`, a template ref, or a custom element. The library is written in TypeScript and ships full type definitions; docs and demos are built with VitePress and tests run under Vitest.

## Production Notes

**SSR and hydration.** Composables that read browser globals return a stable default on the server and update after mount. This means the first client render must match the server's default (e.g. `usePreferredDark()` resolves to its fallback on the server), or you get a hydration mismatch. Guard browser-only logic behind `onMounted` or the composable's own client check; do not read `.value` and branch on it during setup on the server path.

**Bundle size is per-import, but watch add-ons.** `@vueuse/core` tree-shakes cleanly, so importing three functions pulls in three functions. The trap is `@vueuse/integrations`, which re-exports wrappers around axios, focus-trap, qrcode, and similar — a careless barrel import can drag a heavy dependency into the bundle. Import integration composables from their specific subpaths.

**Reactivity footguns.** Destructuring a returned reactive object breaks reactivity unless the function returns refs (most do) — check whether a given composable returns refs or a reactive object before destructuring. Passing a plain value where the function expected a reactive source silently gives you a one-shot, non-updating result.

**Version/Vue coupling is strict and has moved fast.** VueUse tracks Vue's reactivity closely, and major versions raise the minimum Vue: v14 requires Vue 3.5+, v13 requires Vue 3.3+, and v12 dropped Vue 2 support entirely[^2]. Vue 2 users are pinned to the v11.x line, which is maintenance-only. Upgrading across a VueUse major usually means upgrading Vue too, so pin the pair deliberately.

**API stability.** The core, widely-used composables are stable. The long tail — experimental sensors, niche browser-API wrappers — churns more, and individual functions have been renamed or deprecated between majors. Read the changelog for the specific functions you depend on rather than assuming a major bump is drop-in.

## When to Use / When Not

**Use when:**
- You are building a Vue 3 app and repeatedly need reactive wrappers over DOM events, browser APIs, storage, or timers.
- You want SSR-aware, auto-cleaning composables without writing lifecycle plumbing.
- You are on Nuxt and want auto-imported composables via `@vueuse/nuxt`.

**Avoid when:**
- You need one or two trivial helpers (a debounce, a toggle) — a local composable or a plain utility avoids the dependency and the API-surface overhead.
- You are still on Vue 2 and cannot pin to v11.x maintenance releases.
- Your problem is server-state/data-fetching — a dedicated cache library fits better than VueUse's thinner network composables.

## Alternatives

- streamich/react-use — the React library VueUse was modeled on; use it when you are in React, not Vue.
- unjs/vue-composable style hand-rolled composables — use when you need one or two utilities and want zero dependencies.
- TanStack/query — use instead when the real need is server-state caching, deduping, and revalidation rather than reactive browser primitives.
- lodash / es-toolkit — use for pure (non-reactive) data utilities; don't pull in VueUse just for `debounce` or `omit`.
- unjs/radix-vue (reka-ui) — use when you want headless UI components rather than low-level reactive primitives.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-12 | Repository created; early 0.x releases modeled on react-use[^1]. |
| 1.0 | ~2020 | First stable line for the Vue 3 Composition API. |
| 11.x | 2024 | Last line to support Vue 2 (maintenance-only). |
| 12.0 | ~2024 | Vue 2 support dropped; Vue 3 only[^2]. |
| 13.0 | ~2025 | Minimum Vue raised to 3.3+[^2]. |
| 14.0 | 2026 | Minimum Vue raised to 3.5+[^2]. |

## References

[^1]: VueUse README — "This project is heavily inspired by streamich/react-use"; © 2019-PRESENT Anthony Fu. https://github.com/vueuse/vueuse
[^2]: VueUse README install notes — "From v14.0, VueUse requires Vue v3.5+ / From v13.0, Vue v3.3+ / From v12.0, VueUse no longer supports Vue 2." https://github.com/vueuse/vueuse#-install

## Tags

vue, vue3, typescript, composables, composition-api, utility-library, frontend, reactivity, ssr, tree-shakeable, hooks
