# pmndrs/zustand

Zustand — a small, fast, scalable state-management library for React. The modern default for "Redux is too much, Context is too little".

## What it is

A TypeScript state-management library for React based on hooks. No providers needed, no Redux-style action types, no boilerplate. Define a store with `create()`, expose state + actions, consume via a selector hook. Authored by the pmndrs (poimandres) collective; tightly integrated with the broader React Three Fiber / Zustand / Jotai ecosystem.

## Key features

- Zero-boilerplate store creation: `create<T>((set) => ({...}))`.
- Selector-based subscriptions — components re-render only when selected state changes.
- Middleware system: persist, devtools, immer, subscribe-with-selector.
- TypeScript types bundled, strict mode-friendly.
- ~3KB gzipped.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- npm package: `zustand`.

## When to reach for it

- You're a React team and want lighter state management than Redux.
- You want global state that doesn't require Context boilerplate.
- You want a TypeScript-first store with good types.

## When *not* to reach for it

- You're already invested in Redux Toolkit and have working patterns.
- You need server-state — TanStack Query, SWR, Apollo fit that better.
- You need fine-grained reactivity — Jotai or Valtio (same org) may fit.

## Maturity signal

58k stars, 2k forks, MIT, actively maintained. Open issues = 4. The pmndrs collective's reputation + low issue count signals mature maintenance.

## Alternatives

- Redux Toolkit — when you want explicit action / reducer patterns.
- Jotai — atom-based reactivity from the same org.
- TanStack Query — for server state.
- React Context + useReducer — for simple cases.

## Tags

typescript, react, state-management, library, hooks, mit-license, redux
