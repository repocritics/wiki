# pmndrs/valtio

> Proxy-based state management for React and vanilla JS: mutate a plain object, components re-render only on the parts they read.

[GitHub repo](https://github.com/pmndrs/valtio) ·
[Official website](https://valtio.dev) ·
[License: MIT](https://github.com/pmndrs/valtio/blob/main/LICENSE)

## Overview

Valtio is a small state library from the Poimandres (pmndrs) collective, authored by
Daishi Kato — the same maintainer behind zustand, jotai, and react-tracked[^1]. Its
premise is mutation: you wrap a plain object with `proxy()`, then write to it like any
JavaScript object (`state.count++`), and subscribers are notified. There are no actions,
reducers, dispatchers, or immutable-update ceremony. As of 2026 it holds roughly 10.2k
stars with a stable, low-churn release cadence — the last push was mid-2026 and open
issues sit in the single digits, which for a library this widely depended-on signals a
mature, largely finished design rather than neglect.

The defining tension is the split between the mutable proxy and the immutable *snapshot*.
You mutate `state` (the proxy) from anywhere, but in React render you read from
`useSnapshot(state)`, which returns a deeply frozen, render-optimized view. The hook
tracks exactly which properties your component accessed and re-renders only when those
change[^2]. This gives fine-grained reactivity for free, but the "write to the proxy, read
from the snapshot" rule is the single most common source of confusion for newcomers, and
misusing `this` inside proxied methods or destructuring the snapshot at the wrong scope
produces subtle staleness bugs.

Valtio is unopinionated about structure. It ships no folder convention, no middleware
stack, and no async model beyond React Suspense compatibility. That minimalism is
deliberate and is why it composes cleanly with react-three-fiber, react-native, and
non-React runtimes — but it also means best practices (organizing actions, persistence,
context scoping) live in community recipes rather than the core.

## Getting Started

```bash
npm install valtio
```

```jsx
import { proxy, useSnapshot } from 'valtio'

// Wrap any object — it becomes a self-aware proxy.
const state = proxy({ count: 0, text: 'hello' })

// Mutate from anywhere, no setter required.
setInterval(() => { ++state.count }, 1000)

function Counter() {
  // Reads from a snapshot; re-renders only when `count` (not `text`) changes.
  const snap = useSnapshot(state)
  return (
    <div>
      {snap.count}
      <button onClick={() => ++state.count}>+1</button>
    </div>
  )
}
```

Read from `snap` in render; write to `state` everywhere else. That one rule of thumb
covers most correct usage.

## Architecture / How It Works

Valtio is built on two distinct proxy layers, which is worth understanding because the
"two proxies" model explains most of its behavior[^2]:

- **`proxy()` (valtio/vanilla)** — a write-tracking proxy. It intercepts mutations,
  recursively proxies nested objects, batches notifications, and calls subscribers. This
  is the mutable source of truth and works with no React at all.
- **`createProxy()` (from proxy-compare)** — a read-tracking proxy used internally by
  `useSnapshot`. It records which keys a component touched during render, so re-renders
  can be scoped to the accessed paths[^3].

A `snapshot()` is an immutable, structurally-shared copy taken at a point in time; it is
frozen with `Object.freeze`, so attempts to mutate it throw. `useSnapshot` calls
`snapshot()` and wraps the result in the read-tracking proxy. This separation is why you
cannot call a proxied method through the snapshot and mutate through `this` — inside a
snapshot, `this` points at a frozen object. The maintainers recommend arrow functions
closing over `state` instead of `this`-based methods for exactly this reason.

The core exports (`proxy`, `useSnapshot`, `subscribe`, `snapshot`, `ref`) are tiny. Heavier
functionality lives in `valtio/utils`: `subscribeKey`, `watch` (auto-subscribe on read),
`devtools` (Redux DevTools bridge), `proxySet` / `proxyMap` (proxy-backed Set/Map that the
raw proxy can't otherwise track), and `useProxy` (a convenience that returns a
render-and-callback-usable proxy to sidestep the snapshot/proxy split). `ref()` is the
escape hatch for holding an object in state *without* proxying it — necessary for DOM
nodes, class instances, or large nested data you don't want traversed[^4].

Because mutation notifications are batched by default, multiple synchronous writes collapse
into a single re-render. A `{ sync: true }` option on `useSnapshot` opts out of batching,
which is the standard fix for controlled `<input>` cursor-jump problems.

## Production Notes

- **The proxy/snapshot boundary is the footgun.** Reading `state` directly in render (instead
  of `snap`) silently disables render optimization and can miss updates; writing to `snap`
  throws. `eslint-plugin-valtio` exists specifically to catch these and is strongly
  recommended for teams.
- **TypeScript strictness.** `useSnapshot` returns a deeply `readonly` type, which is correct
  but can be too strict for some call sites; the documented workaround is a module
  augmentation loosening the return type (see issue #327). Expect occasional friction passing
  snapshots into APIs typed for mutable objects.
- **Non-plain values need `ref()`.** Storing DOM nodes, class instances, Maps, or Sets
  directly in a proxy either fails to track or breaks; use `ref()`, `proxySet`, or `proxyMap`
  respectively. Forgetting this is a common "why doesn't my state update" report.
- **Suspense de-opt.** Valtio supports React 19's `use` hook for reading promises held in
  state, but the built-in path suffers a de-opt that prevents `useTransition` from working
  well; the third-party `use-valtio` library exists to mitigate it.
- **No built-in persistence, no time-travel by default.** Persistence, hydration, and action
  organization are community recipes, not core features. DevTools integration works only for
  plain objects/arrays.
- **`this` semantics.** Object methods using `this` work on the proxy but not the snapshot.
  The maintainers themselves flag this as expert-only; prefer arrow functions bound to the
  module-level `state`.

## When to Use / When Not

**Use when:**
- You want mutable, low-ceremony state and dislike reducer/action boilerplate.
- You need automatic fine-grained re-render optimization without manual selectors.
- You're outside pure React (react-three-fiber, vanilla JS, react-native) and want one model
  that spans them.
- Your state is a graph of plain objects/arrays that benefits from direct mutation.

**Avoid when:**
- Your team values explicit, traceable state transitions (Redux-style) over implicit mutation.
- You need a large, standardized middleware/persistence ecosystem out of the box.
- Heavy use of class instances, Maps, or non-serializable data would force `ref()` everywhere.
- You want a single, blessed way to structure an app — Valtio deliberately provides none.

## Alternatives

- pmndrs/zustand — same author; hook/store with explicit setters and selectors. Prefer it when you want a small store without proxy magic or immutable-mutation confusion.
- pmndrs/jotai — same author; atomic, bottom-up state. Prefer when state composes from many independent primitives rather than one mutable tree.
- reduxjs/redux-toolkit — Immer-backed "mutable-looking" updates with strict, traceable dispatch. Prefer when auditability and devtools/time-travel matter more than terseness.
- mobxjs/mobx — the older observable/proxy reactivity model with a broader decorator ecosystem. Prefer when you want mature computed/reaction primitives and don't mind a larger surface.
- immerjs/immer — not a store but the mutation-to-immutable primitive; prefer when you only need ergonomic immutable updates inside an existing state container.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2020-11 | First public release under pmndrs; proxy + useSnapshot core[^1]. |
| v1.0 | 2021 | Stabilized API: `proxy`, `useSnapshot`, `subscribe`, `snapshot`, utils. |
| v2.0 | 2024 | Major line; requires React 18+, aligns with concurrent React and the `use` hook[^5]. |

## References

[^1]: Valtio repository and pmndrs collective — Daishi Kato (dai-shi), also author of zustand and jotai. https://github.com/pmndrs/valtio
[^2]: Valtio README, "React via useSnapshot" and internals notes on the two-proxy model. https://github.com/pmndrs/valtio#readme
[^3]: proxy-compare — the read-tracking library used for render optimization. https://github.com/dai-shi/proxy-compare
[^4]: Valtio `ref` usage — issues #61 and #178. https://github.com/pmndrs/valtio/issues/61
[^5]: Valtio README, "Compatibility" — v2 works with React 18 and up. https://github.com/pmndrs/valtio#compatibility

## Tags

typescript, javascript, react, state-management, proxy, reactivity, frontend, pmndrs, vanilla-js, immutable-snapshot
