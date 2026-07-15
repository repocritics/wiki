# nanostores/nanostores

> A sub-kilobyte state manager built from many small atomic stores, with framework adapters for React, Vue, Svelte, Solid, Preact, Lit, and Angular.

[GitHub repo](https://github.com/nanostores/nanostores) ·
[License: MIT](https://github.com/nanostores/nanostores/blob/main/LICENSE)

## Overview

Nano Stores is a state-management core that ships in roughly 340–864 bytes
(minified + brotli, depending on which primitives you import) with zero
dependencies[^1]. It was extracted from the Logux ecosystem by Andrey Sitnik —
the author of PostCSS, Autoprefixer, and Browserslist — and is maintained under
the Evil Martians umbrella[^2]. It is framework-agnostic: the core has no React
or Vue coupling, and each UI binding lives in a separate `@nanostores/*` adapter
package.

The defining design choice is **many small atomic stores instead of one central
tree**. Where Redux or Zustand keep a single object and rely on selectors to
carve out slices, Nano Stores pushes granularity into the store graph itself:
each piece of state is its own store, and a component subscribes only to what it
reads. The payoff is tree-shaking — a route chunk pulls in only the stores its
components touch — and no selector-diffing on every global change. The cost is
wiring: you compose behavior from `atom`, `map`, `computed`, and `effect` by
hand, with no single place to inspect "the whole state."

It fits design-system and multi-framework teams, library authors who cannot
assume a host framework, and apps that want reactive logic to live outside
components. It is not a batteries-included framework; routing, persistence, and
remote data are separate community packages, not core.

## Getting Started

```sh
npm install nanostores
# framework binding is a separate package:
npm install @nanostores/react
```

```ts
// store/users.ts
import { atom, computed } from 'nanostores'

export const $users = atom<User[]>([])

export function addUser(user: User) {
  $users.set([...$users.get(), user])   // replace, never mutate in place
}

export const $admins = computed($users, users => users.filter(u => u.isAdmin))
```

```tsx
// components/Admins.tsx
import { useStore } from '@nanostores/react'
import { $admins } from '../store/users'

export const Admins = () => {
  const admins = useStore($admins)          // re-renders only on $admins change
  return <ul>{admins.map(u => <li key={u.id}>{u.name}</li>)}</ul>
}
```

The `$` prefix on store names is a project convention (it also suits Svelte's
auto-subscription), not a language requirement.

## Architecture / How It Works

The core is a handful of factory functions over a minimal store shape — a value
plus a listener set — rather than a class hierarchy:

- **`atom(initial)`** — the base store. `get()` reads, `set(next)` writes,
  `subscribe(cb)` (fires immediately) and `listen(cb)` (fires on next change)
  observe. Change detection is **reference equality** on the value.
- **`map(initial)`** — a one-level object store adding `setKey(key, value)`;
  `listenKeys` / `subscribeKeys` observe specific keys. Nested keys can be
  watched by base key.
- **`computed(deps, fn)`** — derived store, recomputed on any dependency change.
  **`batched(deps, fn)`** is the same but defers to end-of-tick so multiple
  upstream writes collapse into one recomputation.
- **`effect(deps, fn)`** — runs a side effect across multiple stores with
  cleanup semantics.
- **`batch(fn)`** — groups writes into one transaction; listeners fire at most
  once on the outermost flush, observing final values.

The most distinctive feature is **lazy mount/disabled modes**. A store has two
states: *mounted* (at least one listener) and *disabled* (none). `onMount`
registers setup that runs on first subscription and a teardown that runs after
the last unsubscribe — so a store can open a network connection or URL listener
only while the UI actually uses it. To avoid flicker, the transition to disabled
is deferred by a **1-second delay** after the last listener leaves[^1].

Store lifecycle events (`onStart`, `onStop`, `onSet`, `onNotify`) expose hook
points; `onSet` and `onNotify` receive an `abort()` to veto a change or its
notification. Async initialization is tracked with `task()` / `startTask()` and
drained via `await allTasks()`, which is how SSR and tests wait for data before
rendering. React/Preact adapters build on `useSyncExternalStore`, so tearing and
concurrent-rendering correctness are handled by the framework rather than
reimplemented.

## Production Notes

**Reference-equality change detection is the top footgun.** Because `atom`
compares with `===`, mutating an object or array in place and calling `set()`
with the same reference will not notify listeners. Every update must produce a
new reference (`$users.set([...$users.get(), user])`). Teams migrating from
MobX's mutable model hit this immediately.

**Stores are module-level singletons — dangerous on the server.** A store
declared at module scope is shared across every SSR request in the same process.
Without care, one user's data leaks into another's render. The intended pattern
is to hydrate per request, drain `allTasks()`, and use `cleanStores()` between
renders in tests; production SSR needs an explicit strategy to isolate or reset
state per request rather than relying on module globals.

**ESM-only distribution.** Nano Stores ships as ES modules; the README keeps a
standing "Known Issues → ESM" note. CommonJS toolchains and older Jest setups
need `transformIgnorePatterns` / transpilation config or they fail to import.
Vitest and modern bundlers handle it out of the box.

**`get()` outside tests causes stale reads.** Calling `store.get()` in render
reads a value without subscribing, so the component won't re-render on change —
the project's own best practices say to avoid `get()` outside tests and use
`useStore()`. The 1-second disable delay is also observable: a store stays
mounted for a second after its last unsubscribe (`keepMount()` / `cleanStores()`
exist to make lazy initializers deterministic in tests), which can briefly keep
connections open after unmount.

**The ecosystem is many separate repos.** Router, persistent, query, i18n,
async-computed, logger, and every framework binding are independently versioned
packages. This keeps the core tiny but means maturity, release cadence, and
TypeScript quality vary per package rather than moving as one release.

## When to Use / When Not

**Use when:**
- You want state logic that outlives any single component and is framework-portable (a design system, an SDK, a multi-framework monorepo).
- Tree-shaking and bundle size are first-order concerns.
- You prefer fine-grained atomic stores over one global tree plus selectors.
- You want lazy state that only does work (connections, subscriptions) while mounted.

**Avoid when:**
- You want centralized state with time-travel devtools and a single inspectable tree — Redux Toolkit fits better.
- Your team relies on mutable, proxy-based reactivity — MobX or Vue's reactivity is more ergonomic.
- You need a batteries-included data/router/cache stack in core rather than assembling community packages.
- Your toolchain is hard-locked to CommonJS and you can't configure ESM interop.

## Alternatives

- pmndrs/zustand — single hook-based store, React-first; simpler mental model, larger surface, no framework-agnostic core.
- pmndrs/jotai — atomic like Nano Stores but React-only and Provider/Context-based; use it when you're all-in on React and want atom composition tied to the component tree.
- preactjs/signals — comparably tiny reactive primitive; use it when signal-style `.value` reactivity and Preact/React integration are the priority.
- reduxjs/redux-toolkit — use when you need a centralized store, mature devtools, and strong middleware/time-travel conventions.
- mobxjs/mobx — use when you want transparent mutable state with automatic dependency tracking instead of explicit immutable `set()`.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-10 | Extracted from the Logux ecosystem by Andrey Sitnik[^2]. |
| 0.x line | 2021–2024 | Core primitives (`atom`, `map`, `computed`), lazy mount, framework adapters added incrementally. |
| `batched` / `effect` / `batch` | added over 0.9–0.11 era | Deferred derived stores, multi-store effects, and write transactions. |
| last commit | 2026-07 | Actively maintained; ~7.5k stars, low open-issue count. |

## References

[^1]: nanostores/nanostores README — size, lazy mount/disabled modes, and API guide. https://github.com/nanostores/nanostores
[^2]: Evil Martians open-source, and Andrey Sitnik (author of PostCSS, Autoprefixer, Browserslist, Logux). https://evilmartians.com/opensource

## Tags

state-management, typescript, react, vue, svelte, atomic-stores, tree-shaking, frontend, reactive, framework-agnostic, esm
