# preactjs/signals

> Fine-grained reactive state primitives — a signal graph bolted onto Preact natively and onto React through a compiler.

[GitHub repo](https://github.com/preactjs/signals) ·
[Announcement post](https://preactjs.com/blog/introducing-signals/) ·
[License: MIT](https://github.com/preactjs/signals/blob/main/LICENSE)

## Overview

Signals is the Preact team's implementation of fine-grained reactivity: a value
container (`signal`) whose reads are tracked, so any computation or component
that reads it is re-run automatically when it changes. The idea is not new —
Preact Signals is a deliberate port of the reactivity model popularized by
Solid, Vue, and MobX into the Preact/React world — but its selling point is a
very small core (a few kB) and, in Preact, integration deep enough that a signal
read inside JSX can update a single DOM text node without re-rendering the
component at all[^1].

The defining tension is React. Preact controls its own renderer, so `@preact/signals`
can subscribe components and bind text nodes directly. React exposes no such
hooks, so `@preact/signals-react` has gone through two eras: an early version
that monkey-patched React's internal dispatcher (fragile across React versions),
and the current approach, a Babel/SWC transform that rewrites components to call
a `useSignals()` hook so they subscribe correctly[^2]. This split — elegant on
Preact, compiler-assisted and caveat-laden on React — is the single most
important thing to understand before adopting it.

As of 2026 the project is small but actively maintained (last pushed within a
day of this writing; ~4.5k stars, ~120 forks — a focused single-purpose library
rather than a framework). The same team drives the TC39 Signals proposal, which
would eventually standardize this primitive in JavaScript itself[^3].

## Getting Started

```bash
# Preact
npm install @preact/signals
# React (also install the transform below)
npm install @preact/signals-react
# Framework-agnostic core only
npm install @preact/signals-core
```

```jsx
// Preact — the component never re-renders; the text node updates directly
import { signal, computed } from "@preact/signals";

const count = signal(0);
const double = computed(() => count.value * 2);

function Counter() {
  return (
    <button onClick={() => count.value++}>
      {count} × 2 = {double}
    </button>
  );
}
```

```js
// Framework-agnostic core
import { signal, computed, effect, batch } from "@preact/signals-core";

const a = signal(1);
const b = signal(2);
const sum = computed(() => a.value + b.value);

effect(() => console.log("sum:", sum.value)); // logs "sum: 3"
batch(() => { a.value = 10; b.value = 20; });  // one recompute → logs "sum: 30"
```

## Architecture / How It Works

The core is a lazy, pull-based dependency graph with push notification. Reading
`signal.value` inside a `computed` or `effect` registers a dependency edge;
writing a new value marks dependents dirty but does not eagerly recompute them.
A `computed` recomputes only when it is actually read and one of its sources has
changed, using per-node version counters to skip work when nothing meaningful
moved. This makes the graph glitch-free (no intermediate/inconsistent values are
observed) and avoids the recomputation storms that naive observer patterns
produce[^1]. `signal.peek()` reads without subscribing; `untracked(fn)` runs a
block without registering dependencies; `batch(fn)` coalesces multiple writes
into a single notification pass.

The Preact binding (`@preact/signals`) hooks the renderer directly: it augments
Preact's diff so that a component reading `signal.value` subscribes, and a signal
placed directly in JSX (`{count}`) is bound to a raw text node — updates bypass
virtual-DOM diffing entirely. Preact-specific helpers (`useSignal`,
`useComputed`, `useSignalEffect`, `<Show>`, `<For>`) round out the integration.

The React binding cannot touch the renderer. `@preact/signals-react` relies on
`@preact/signals-react-transform`, a Babel/SWC plugin that detects components
using signals and injects a `useSignals()` call so re-renders are tracked
through React's normal reconciliation[^2]. Without the transform you must call
`useSignals()` (or use the `<Signal/>`-style helpers) manually, and mistakes
show up as components that silently fail to update. This is the coupling story:
on Preact the reactivity is native; on React it is a build-time rewrite plus a
hook, and it re-renders the whole component (it does not do Preact's direct
text-node binding).

## Production Notes

- **The React transform is not optional in practice.** Relying on the
  auto-tracking behavior means every consumer app needs the Babel/SWC plugin
  wired into its build. Toolchains that resist custom Babel config (some Vite
  setups, Next.js with the SWC compiler) require the matching plugin variant;
  getting it wrong yields components that don't re-render.
- **The old internals-patching era is a real hazard.** `@preact/signals-react`
  before the transform patched React's dispatcher and was sensitive to React
  minor versions and to StrictMode double-invocation. If you find old tutorials
  or a pinned old version, expect breakage on React upgrades — move to the
  transform-based version[^2].
- **Signals are external, mutable, module-level state.** That is convenient but
  bypasses React's data-flow assumptions: values living outside the component
  tree interact awkwardly with SSR hydration, Suspense, and concurrent
  rendering. For SSR you must be careful not to share a signal instance across
  requests (module-scope singletons leak state between users); the React package
  documents SSR-specific usage that must be followed.
- **`.value` is easy to forget.** Reading `count` vs `count.value` is a common
  bug: passing the signal object where you meant its value (or vice versa) fails
  quietly. In JSX the bare signal renders; in logic you almost always want
  `.value`.
- **Ecosystem is smaller than Zustand/Jotai/Redux.** Fewer middlewares,
  devtools, and integration recipes. A debug package exists, but tooling depth
  and Stack Overflow coverage are thinner than the mainstream React state
  options.
- **Interop with React libraries.** Because updates flow outside React state,
  some libraries expecting `useState`/`useReducer` semantics (form libs, certain
  memoization patterns) need adapters or manual bridging.

## When to Use / When Not

**Use when:**
- You're on Preact and want the smallest, most direct reactive state model.
- You want fine-grained updates and are comfortable adding a build-time transform
  to your React app.
- You prefer signal graphs (computed/effect) over selector-based stores.
- You want an API close to the future TC39 Signals standard.

**Avoid when:**
- You can't add a Babel/SWC plugin to your React build pipeline.
- You lean heavily on React Server Components / Suspense / concurrent features
  and want state that composes with them natively.
- You want the largest ecosystem, devtools, and hiring pool — Zustand/Redux win
  on maturity.
- You want state that lives inside the React tree rather than in module scope.

## Alternatives

- solidjs/solid — signals are the entire framework's foundation; use instead when you can leave React and want reactivity native to the renderer with no transform.
- pmndrs/jotai — atomic state for React; use instead when you want React-idiomatic atoms with Suspense support and no build-time plugin.
- pmndrs/zustand — minimal external store; use instead when you want a simple hook-based store without a signal graph or compiler.
- mobxjs/mobx — mature observable reactivity; use instead when you want a batteries-included reactive store with a large ecosystem and devtools.
- tc39/proposal-signals — the standards-track primitive from the same authors; watch instead when you'd rather wait for a built-in than adopt a library.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repo created | 2022-08-08 | Initial development on GitHub[^4]. |
| Public announcement | 2022-09 | "Introducing Signals" post; core + Preact + React packages[^1]. |
| React transform era | 2023 | `@preact/signals-react` moved from patching React internals to a Babel/SWC transform + `useSignals()`[^2]. |
| TC39 Signals proposal | 2024 | Same maintainers propose a standardized signal primitive for JS[^3]. |

## References

[^1]: Preact team, "Introducing Signals" — announcement of the reactivity model, core algorithm, and direct DOM text binding. https://preactjs.com/blog/introducing-signals/
[^2]: `@preact/signals-react` package README — the `useSignals` hook, the `@preact/signals-react-transform` Babel/SWC plugin, and React SSR usage/limitations. https://github.com/preactjs/signals/tree/main/packages/react
[^3]: TC39 Signals proposal. https://github.com/tc39/proposal-signals
[^4]: GitHub repository metadata, preactjs/signals. https://github.com/preactjs/signals

## Tags

typescript, javascript, state-management, reactivity, signals, preact, react, frontend, fine-grained-reactivity, library
