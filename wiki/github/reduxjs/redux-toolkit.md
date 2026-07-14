# reduxjs/redux-toolkit

> The official, opinionated wrapper around Redux — turns Redux's famous boilerplate into a few slice definitions, and bundles a data-fetching layer (RTK Query) on top.

[GitHub repo](https://github.com/reduxjs/redux-toolkit) ·
[Official website](https://redux-toolkit.js.org) ·
[License: MIT](https://github.com/reduxjs/redux-toolkit/blob/master/LICENSE)

## Overview

Redux Toolkit (RTK, published as `@reduxjs/toolkit`) is the Redux team's answer to the three complaints that dogged classic Redux: the store is too complicated to configure, you need too many extra packages, and it demands too much boilerplate. It started life as `redux-starter-kit` and was renamed at its 1.0 release in 2019[^1]. Today the Redux docs actively discourage hand-writing raw Redux; RTK is presented as the default and only recommended way to use Redux[^2].

The defining tradeoff is **opinionation in exchange for magic**. RTK ships `immer` so you write "mutative" code (`state.value += 1`) that is actually applied immutably, and it wires `redux-thunk`, the Redux DevTools connection, and development-only invariant checks into `configureStore` by default. This removes enormous ceremony, but it also hides several moving parts behind convention — the immutable-update rewrite, the serializability checks, and the middleware pipeline are all invisible until one of them fails at runtime.

The second thing in the box is **RTK Query**, an optional data fetching and caching layer that lives at separate entry points. It overlaps heavily in purpose with TanStack Query, but is built on the Redux store so its cache is inspectable in Redux DevTools. Whether you want RTK Query specifically — versus a server-state library that does not require Redux at all — is the main architectural decision when adopting the package.

## Getting Started

```bash
npm install @reduxjs/toolkit react-redux
# scaffold a fresh Vite + TS app pre-wired for Redux:
npx degit reduxjs/redux-templates/packages/vite-template-redux my-app
```

```ts
// counterSlice.ts
import { createSlice, configureStore, type PayloadAction } from '@reduxjs/toolkit'

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    // Looks mutative — Immer applies it immutably under the hood.
    increment: (state) => { state.value += 1 },
    addBy: (state, action: PayloadAction<number>) => { state.value += action.payload },
  },
})

export const { increment, addBy } = counterSlice.actions

export const store = configureStore({
  reducer: { counter: counterSlice.reducer },
})

store.dispatch(increment())
store.dispatch(addBy(5))
console.log(store.getState().counter.value) // 6
```

## Architecture / How It Works

RTK is a thin set of factory functions layered on the unchanged Redux core. The pieces:

- **`configureStore()`** wraps the core `createStore`/`configureStore` internals with sensible defaults: it calls `combineReducers` for you when you pass a reducer map, applies `redux-thunk`, connects the DevTools Extension, and in development inserts two middleware — `immutableStateInvariant` (throws if you accidentally mutate state outside Immer) and `serializableStateInvariant` (warns when non-serializable values enter the store or an action).
- **`createSlice()`** is the centerpiece. It combines `createReducer` + `createAction`: you give it a name, initial state, and a map of case reducers, and it generates the reducer, the action creators, and the action-type strings together. Each case reducer runs inside an Immer `produce`, so the draft can be mutated directly.
- **`createAsyncThunk()`** wraps a promise-returning function into a thunk that auto-dispatches `pending` / `fulfilled` / `rejected` actions; you handle those in a slice's `extraReducers` builder.
- **`createEntityAdapter()`** generates reducers and memoized selectors for normalized collections (`{ ids: [], entities: {} }`), the recommended shape for relational data.
- **`createListenerMiddleware()`** is a lightweight reactive middleware — run an effect when a matching action is dispatched or state changes — pitched as a smaller replacement for redux-saga / redux-observable.
- **`createSelector()`** is re-exported straight from Reselect for memoized derived state.

**RTK Query** sits above all this. `createApi()` defines an "API slice": a set of endpoints, a `baseQuery` (usually `fetchBaseQuery`, a thin `fetch` wrapper), and a tag system. The React entry point (`@reduxjs/toolkit/query/react`) code-generates hooks (`useGetPostsQuery`, `useAddPostMutation`) from the endpoint definitions. Under the hood it is entirely a reducer + middleware + generated selectors — the cache is a normal slice of Redux state, keyed by endpoint name plus a serialized form of the query arguments. Invalidation is tag-based: mutations declare which tags they invalidate, and queries providing those tags refetch.

The coupling story: RTK depends on `immer`, `reselect`, and `redux-thunk`, and re-exports much of the Redux core, so adding RTK effectively pins those versions. RTK Query is optional dead-code-eliminated weight — if you never import from `/query`, it should tree-shake out.

## Production Notes

**The serializability check catches real bugs but also fires on legitimate values.** By default `configureStore` warns (dev only) when you dispatch actions or store state containing `Date`, `Map`, `Set`, class instances, or promises. Storing a `Date` object or a non-serializable payload is the single most common first-week RTK surprise. The fix is either to serialize at the boundary or to configure `serializableCheck` to ignore specific paths/action types — commonly needed for redux-persist, which dispatches non-serializable actions.

**Immer's "mutate or return, not both" rule.** Inside a `createSlice` reducer you may mutate the draft *or* return a brand-new state object, but doing both in one reducer throws. This trips people writing `state.x = 1; return state` or conditionally returning early. Also, replacing the whole state requires `return newState` — reassigning the `state` parameter does nothing.

**Immer overhead on large state.** Immer's structural sharing has a per-update cost. For very large arrays/objects updated in hot paths, the invariant middleware in development can dominate render time; the immutability and serializability checks walk the entire state tree on every action. Both are dev-only, but large stores may still want to tune or disable them locally.

**RTK Query cache lifetime is subscription-based, not time-based.** Cached data for an endpoint is removed a short interval (`keepUnusedDataFor`, default 60 seconds) after the last component subscribing to it unmounts. Teams expecting a persistent client cache are surprised when navigating away and back triggers a refetch. Tune `keepUnusedDataFor` per endpoint, and understand that arg serialization determines cache-key identity — passing a new object literal as args each render can fragment the cache.

**One API slice per base URL.** `createApi` is intended to be called roughly once per backend. Splitting one logical API across many `createApi` calls loses cross-endpoint tag invalidation. For large apps use `injectEndpoints` to code-split endpoints into a single API slice.

**Upgrade pain — v1 → v2 (Dec 2023).** RTK 2.0 removed the object-map syntax for `extraReducers` and `createReducer`; the builder-callback form (`builder.addCase(...)`) is now mandatory[^3]. It also modernized packaging (ESM-first dual package) and bumped Redux core to 5.0, Immer to 10, and Reselect to 5, each with their own minor breaking changes. Codemods were provided, but the object-syntax removal broke a lot of older tutorials' code.

## When to Use / When Not

**Use when:**
- You are already committed to Redux and want to stop hand-writing action types, action creators, and switch-statement reducers.
- You want time-travel debugging and a globally inspectable state tree via Redux DevTools.
- You need a single store that mixes complex client state (wizards, undo, cross-cutting derived data) with server cache, and want both visible in one DevTools timeline.
- Your team already knows Redux idioms and the learning cost is sunk.

**Avoid when:**
- Your state is mostly server data. A dedicated server-state library (TanStack Query, SWR) is lighter and does not require Redux at all.
- You want minimal global state with no provider ceremony — Zustand or Jotai are far smaller.
- Bundle size is critical and you would only use a fraction of RTK — the core plus Immer/Reselect is non-trivial weight.
- The app is small; `useState` + Context may be all you need.

## Alternatives

- pmndrs/zustand — use instead when you want minimal, hook-based global state with no provider, reducers, or DevTools ceremony.
- pmndrs/jotai — use instead when your state decomposes naturally into small independent atoms rather than one big tree.
- TanStack/query — use instead of RTK Query when your problem is purely server-state caching and you are not otherwise on Redux.
- mobxjs/mobx — use instead when you prefer transparent observable/reactive state over explicit dispatched actions.
- statelyai/xstate — use instead when flows are complex enough to warrant explicit state machines and statecharts.

## History

| Version | Date | Notes |
|---------|------|-------|
| redux-starter-kit 0.x | 2018–2019 | Original package name; the boilerplate-reduction experiment[^1]. |
| 1.0 | 2019-10 | Renamed to `@reduxjs/toolkit`; `createSlice`/`configureStore` stabilized[^1]. |
| 1.6 | 2021 | RTK Query added as optional data-fetching/caching layer[^4]. |
| 1.8 | 2022 | `createListenerMiddleware` added as a saga/observable alternative. |
| 2.0 | 2023-12 | Builder-callback syntax required; ESM-first packaging; Redux 5 / Immer 10 / Reselect 5[^3]. |

## References

[^1]: Redux Toolkit docs, "Redux Toolkit: Overview / history". https://redux-toolkit.js.org/introduction/getting-started
[^2]: Redux docs, "Redux Essentials — Redux Toolkit is the recommended approach." https://redux.js.org/introduction/why-rtk-is-redux-today
[^3]: Redux Toolkit docs, "Migrating to RTK 2.0 and Redux 5.0." https://redux-toolkit.js.org/usage/migrating-to-modern-redux
[^4]: RTK Query overview. https://redux-toolkit.js.org/rtk-query/overview

## Tags

typescript, javascript, redux, state-management, react, data-fetching, rtk-query, immer, frontend, caching
