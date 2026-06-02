# reduxjs/redux

Redux — the predictable global state-management library that taught a generation of JS developers about reducers, actions, and immutable state. Still widely used; complemented by Redux Toolkit as the modern way to use it.

## What it is

A TypeScript library that codifies a unidirectional data-flow pattern: state lives in a single store, actions are plain objects describing intent, reducers are pure functions that produce next state from previous state + action. Inspired by Elm + Flux. Redux Toolkit (RTK) is the canonical modern API that hides the verbosity Redux became known for.

## Key features

- Single store, immutable state, pure reducers.
- Middleware system (`redux-thunk`, `redux-saga`, RTK Query).
- Time-travel debugging via Redux DevTools.
- Redux Toolkit — the modern API with reducer generation, async thunks, RTK Query.
- TypeScript types bundled.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- npm packages: `redux`, `@reduxjs/toolkit`, `react-redux`.

## When to reach for it

- You're maintaining an existing Redux codebase.
- You want predictable, time-travel-debuggable state.
- You're building large apps where the structure pays off.

## When *not* to reach for it

- You're starting fresh and want lighter — Zustand, Jotai, TanStack Query for most use cases.
- Simple app — plain useState / useReducer is enough.

## Maturity signal

61k stars, 15k forks, MIT, actively maintained. 10+ years.

## Alternatives

- Zustand — lighter modern alternative.
- TanStack Query — for server state specifically.
- Jotai — atom-based.

## Tags

javascript, typescript, react, state-management, library, mit-license, redux
