# pmndrs/jotai

> Atomic, bottom-up state management for React — state lives in tiny `atom` objects instead of one central store.

[GitHub repo](https://github.com/pmndrs/jotai) ·
[Official website](https://jotai.org) ·
[License: MIT](https://github.com/pmndrs/jotai/blob/main/LICENSE)

## Overview

Jotai is a state-management library for React built by Daishi Kato under the
Poimandres (pmndrs) collective, first published in 2020[^1]. Its model is
"bottom-up": you declare many small, independent `atom`s and compose them,
rather than defining one large store and slicing into it (the Redux/Zustand
"top-down" model). The design is openly inspired by Meta's Recoil but drops
Recoil's string-key requirement — atoms are keyed by object identity, not by a
registered name — which removes a whole class of key-collision and
serialization concerns[^2].

The core is intentionally small (a couple of kilobytes) and does almost
nothing on its own: `atom()` returns a config object, not a value. Values only
exist inside a store, and a component reads them with `useAtom`, which
subscribes it to exactly that atom and its computed dependencies. The selling
point is fine-grained re-rendering: updating one atom re-renders only the
components that read it, without the manual selector/memo discipline that
React Context or a monolithic store demand.

The defining tension is the atomic model itself. Fine-grained subscriptions and
composable derived atoms are elegant for local and mid-size state, but a large
app ends up with hundreds of module-level atom constants and a dependency graph
that is implicit in code rather than visible in one place. Teams that want a
single inspectable state tree with time-travel (the Redux value proposition)
often find Jotai's diffuse graph harder to reason about at scale.

## Getting Started

```bash
npm i jotai
```

```jsx
import { atom, useAtom } from 'jotai'

// atoms are module-level constants — never create them inside render
const countAtom = atom(0)
const doubledAtom = atom((get) => get(countAtom) * 2) // derived, read-only

function Counter() {
  const [count, setCount] = useAtom(countAtom)
  const [doubled] = useAtom(doubledAtom)
  return (
    <button onClick={() => setCount((c) => c + 1)}>
      {count} (x2 = {doubled})
    </button>
  )
}
```

A writable derived atom takes a read and a write function:

```jsx
const decrementAtom = atom(
  (get) => get(countAtom),
  (get, set) => set(countAtom, get(countAtom) - 1),
)
```

## Architecture / How It Works

An `atom` is a plain config object — a definition with a read function and
(optionally) a write function. It holds no value. Values live in a **store**,
which maps each atom config to its current state using a `WeakMap` keyed on the
atom's object identity. Because the key is the object, two `atom(0)` calls
create two distinct atoms even though their initial values match. Because the
map is weak, an atom's state is garbage-collected once nothing references the
config.

Reading is what builds the dependency graph. When a derived atom's read function
calls `get(otherAtom)`, the store records that dependency dynamically, on every
evaluation. Dependencies are therefore not declared up front and can change
between reads (conditional `get` calls produce a conditional graph). When a
source atom changes, the store walks its dependents, recomputes them, and
notifies only the subscribed components.

A `Provider` scopes a store to a subtree; without one, Jotai uses a default
module-level store, so `Provider` is optional for simple apps. Multiple
`Provider`s give isolated state trees (useful for SSR and per-request state).

`jotai/utils` ships the pieces most apps need: `atomWithStorage`
(localStorage/AsyncStorage-backed), `atomFamily` (parameterized atoms),
`selectAtom` (memoized slice), `splitAtom` (array of atoms), `loadable`
(non-suspending async), and `atomWithReset`. Async is first-class: an atom whose
read returns a Promise integrates with React Suspense, and its dependents can
`await` it via `get`.

The 1.x → 2.x transition (2023) reworked the store API and async semantics: a
`createStore`/`get`/`set` interface usable outside React, and atoms resolving
synchronously by default with promises surfaced explicitly. Code written for
Jotai 1 that relied on the old async-unwrapping behavior needs review when
upgrading[^3].

## Production Notes

**Never create atoms during render.** `atom()` returns a new identity each
call, so `const a = atom(0)` inside a component makes a fresh, empty atom every
render — usually an infinite update loop or lost state. Atoms belong at module
scope; use `atomFamily` or `useMemo` only when you genuinely need per-instance
atoms.

**Duplicate Jotai copies break silently.** The default store lives in the
`jotai` module instance. If a monorepo or a dependency bundles a second copy of
`jotai`, you get two default stores and components read the "wrong" one with no
error. Deduplicate `jotai` to a single version (peer-dep, resolution, or import
map) — this is one of the most common "state isn't updating" reports.

**Debugging is opaque without help.** Atoms have no names at runtime; a store is
a `WeakMap` of anonymous objects. Set `atom.debugLabel` and use
`jotai-devtools` to get readable graphs, otherwise a large app's state is hard
to inspect. There is no built-in single-tree snapshot the way Redux DevTools
offers.

**SSR and hydration.** `atomWithStorage` reads from storage on the client, which
can mismatch server-rendered HTML; hydrate carefully (the util exposes options
to defer/skip initial reads). Server rendering generally needs an explicit
per-request `Provider` and initial-value hydration via `useHydrateAtoms`.

**Async atoms need Suspense discipline.** A suspending atom throws a promise; a
component reading it must sit under a `<Suspense>` boundary, and an unhandled
rejection propagates to the nearest error boundary. Use `loadable` when you want
loading/error state inline instead of suspending.

**Upgrade friction.** The 1→2 store/async change is the notable one; pin and
read the migration guide before bumping across that boundary. Within 2.x the API
has been stable.

## When to Use / When Not

**Use when:**
- You want fine-grained re-renders without hand-writing selectors/memoization.
- State is naturally composed from many small, independent pieces.
- You need shared state but want to avoid Context re-render fan-out.
- You want an optional-Provider, minimal-boilerplate React-only solution.

**Avoid when:**
- You want a single inspectable state tree with mature time-travel devtools
  (Redux Toolkit fits that better).
- You are not on React — Jotai is React-coupled by design.
- Your team prefers mutable, proxy-style updates (Valtio) or a single top-down
  store (Zustand).
- Most of your "state" is server data — a data-fetching cache is a better fit
  than atoms for that concern.

## Alternatives

- pmndrs/zustand — same authors; single top-down store with hook selectors. Use instead when you prefer one central store over many atoms.
- pmndrs/valtio — proxy-based mutable state. Use instead when you want to mutate objects directly and skip the atomic model.
- facebookexperimental/Recoil — the original atomic library Jotai was inspired by. Effectively unmaintained; prefer Jotai for new atomic-model projects.
- reduxjs/redux-toolkit — centralized, predictable, strong devtools. Use instead when auditability and time-travel matter more than boilerplate.
- TanStack/query — server-state cache. Use instead (or alongside) when the state is remote data rather than client UI state.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2020 | Initial release; core `atom` / `useAtom` model[^1]. |
| 1.0 | 2022 | First stable major; API consolidation[^3]. |
| 2.0 | 2023 | New `createStore` store API, revised async semantics[^3]. |

## References

[^1]: Jotai repository and package history, pmndrs/jotai. https://github.com/pmndrs/jotai
[^2]: Jotai documentation, comparison and core concepts. https://jotai.org/docs/basics/concepts
[^3]: Jotai v2 migration guide. https://jotai.org/docs/guides/migrating-to-v2-api

## Tags

react, state-management, atomic-state, typescript, hooks, frontend, pmndrs, javascript, client-state, suspense
