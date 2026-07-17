# TanStack/store

> Framework-agnostic, type-safe reactive store primitive — the shared state core underneath TanStack Query, Form, Router, and Pacer.

[GitHub repo](https://github.com/TanStack/store) ·
[Official website](https://tanstack.com/store) ·
[License: MIT](https://github.com/TanStack/store/blob/main/LICENSE)

## Overview

TanStack Store is a small immutable-state container with a reactive dependency graph and thin per-framework adapters. It is deliberately low-level: a `Store` holds one immutable value, `Derived` computes values from other stores/deriveds with automatic dependency tracking, `Effect` runs side effects when dependencies change, and `batch` coalesces updates. There is no reducer boilerplate, no action/dispatch layer, and no built-in async or middleware — those are left to the code (or library) built on top.

Its real significance is as infrastructure rather than as an app-developer's first choice. Store was extracted from patterns repeated across the TanStack projects and is now the reactive engine those libraries share[^1]. Most people ship it transitively — via `@tanstack/react-form` or `@tanstack/react-router` — without ever importing `@tanstack/store` directly. Used standalone it competes in the "tiny state manager" niche, but it is less known there than Zustand or Jotai and its documentation is comparatively thin.

The defining tension: Store is a primitive, not a batteries-included solution. You get framework-agnostic reactivity and a genuinely small bundle, but you assemble your own conventions for structuring state, deriving it, and organizing updates. It also remains pre-1.0 (0.x) as of 2026[^2], so minor version bumps can carry API changes.

## Getting Started

```bash
npm install @tanstack/store
# plus one adapter for your framework:
npm install @tanstack/react-store   # or -solid-store, -vue-store, -svelte-store, -angular-store, -lit-store
```

```tsx
import { Store } from "@tanstack/store";
import { useStore } from "@tanstack/react-store";

const countStore = new Store({ count: 0 });

function increment() {
  // updater receives the previous state; return a new immutable value
  countStore.setState((s) => ({ ...s, count: s.count + 1 }));
}

function Counter() {
  // selector subscribes only to the slice you read
  const count = useStore(countStore, (s) => s.count);
  return <button onClick={increment}>Count: {count}</button>;
}
```

Derived (computed) values track their dependencies and must be mounted to become active:

```ts
import { Store, Derived } from "@tanstack/store";

const count = new Store(1);
const double = new Derived({ deps: [count], fn: () => count.state * 2 });
const unmount = double.mount(); // starts recomputation; call to tear down
```

## Architecture / How It Works

At the core is an immutable `Store<T>`. `setState` accepts either a value or an `(prev) => next` updater, replaces `store.state`, and notifies subscribers. State is never mutated in place; the store compares old/new (with an optional `updateFn`) and fans notifications out through a dependency graph rather than blindly re-running every subscriber.

`Derived` and `Effect` are nodes in that graph. A `Derived` declares its `deps` (an array of stores or other deriveds) and a `fn`; when a dep changes, the derived recomputes. Crucially, deriveds and effects have a **mount lifecycle** — `.mount()` registers the node on the graph and returns a teardown function. Forget to mount a `Derived` and it simply never updates; this is a common first-time surprise because nothing errors, the value just goes stale. `batch(() => { ... })` groups multiple `setState` calls so dependents recompute once at the end instead of per-write, which matters when several stores feed one derived.

The framework adapters are thin. Each wraps the core's `subscribe` into that framework's reactivity: `@tanstack/react-store` exposes `useStore(store, selector)` built on `useSyncExternalStore` so React renders are tearing-safe and selector-scoped; the Solid, Vue, Svelte, Angular, and Lit adapters bridge to signals, refs, stores, and reactive controllers respectively. Because all state logic lives in the framework-agnostic core, the same store can be shared across a React island and a Solid island, or unit-tested with no framework at all. The tradeoff is that adapter surface area varies — the React adapter is the most exercised (Form and Router lean on it), and less-used adapters get less battle-testing.

## Production Notes

- **Selectors are your re-render control.** `useStore(store, (s) => s.slice)` only triggers a render when the selected value changes by reference/equality. Returning a fresh object/array from a selector every call (e.g. `(s) => ({ ...s.thing })`) defeats this and can cause render loops; select primitives or memoize.
- **Immutability is not enforced.** The core trusts you to return new values from `setState`. Mutating `store.state` directly, or returning the same reference after an in-place change, will silently skip notifications. There is no dev-mode freeze guard by default.
- **The mount lifecycle is a footgun.** `Derived`/`Effect` do nothing until `.mount()` is called, and you are responsible for invoking the returned unmount to avoid leaking graph nodes in long-lived apps. Framework adapters that create deriveds usually manage this for you; hand-rolled graphs do not.
- **Pre-1.0 churn.** The package is 0.x. Pin exact versions in libraries that depend on it, and read release notes before bumping — signatures like the `Derived` options object and adapter hooks have moved between minor versions historically. Because Form/Router/Query bundle their own `@tanstack/store` range, apps can end up with duplicate copies if versions diverge; dedupe to avoid two separate graphs.
- **No async, persistence, or devtools in core.** There is no built-in async status, middleware, storage, or time-travel. TanStack DevTools can surface some state, but if you need Redux-style tooling you are building it yourself.
- **Bundle cost is genuinely small.** The core is a few kilobytes minzipped, which is much of why the surrounding libraries adopted it instead of pulling in a general-purpose state manager.

## When to Use / When Not

**Use when:**
- You are building a library or design system that needs framework-agnostic reactive state usable from React, Solid, Vue, Svelte, Angular, or Lit with one core.
- You want a tiny primitive and are comfortable defining your own state-shape and update conventions.
- You already live in the TanStack ecosystem and want your app state to compose with Form/Router internals.

**Avoid when:**
- You want an opinionated, documented app state manager out of the box — Zustand or Jotai are more direct and better-trodden for that.
- You need built-in async/server-state handling — that is TanStack Query's job, not Store's.
- You need a stable, semver-guaranteed API today; 0.x means accepting occasional breaking changes.

## Alternatives

- pmndrs/zustand — reach for this when you want a small, well-documented standalone React store without a mount/graph model to manage.
- pmndrs/jotai — use when atomic, bottom-up derived state and Suspense integration fit your app better than a single container.
- reduxjs/redux-toolkit — use when you need strict action/reducer traceability, middleware, and mature devtools for a large team.
- TanStack/query — use instead (or alongside) when your "state" is really server data needing caching, revalidation, and async status.
- solidjs/solid — use its native signals directly when you are Solid-only and don't need cross-framework portability.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2023-08-30 | Repository created; extracted as the shared reactive core for TanStack libraries[^1]. |
| 0.x | 2023–2026 | Iterated pre-1.0 alongside adapters for React, Solid, Vue, Svelte, Angular, and Lit; API still stabilizing[^2]. |
| 0.x | 2026-07 | Actively maintained (last push 2026-07-14); 865 stars, 112 forks, MIT[^2]. |

## References

[^1]: TanStack Store documentation and overview. https://tanstack.com/store
[^2]: TanStack/store repository, GitHub. https://github.com/TanStack/store

## Tags

typescript, javascript, state-management, reactive, store, framework-agnostic, react, vue, svelte, solid, tanstack, frontend
