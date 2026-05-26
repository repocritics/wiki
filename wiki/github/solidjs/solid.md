# solidjs/solid

> Simple and performant reactivity for building user interfaces — fine-grained signals with JSX.

[GitHub repo](https://github.com/solidjs/solid) ·
[Official website](https://www.solidjs.com) ·
[License: MIT](https://github.com/solidjs/solid/blob/main/LICENSE)

## Overview

Solid is a UI library that uses JSX for templates but compiles it to direct DOM operations rather than a virtual DOM, paired with fine-grained signal-based reactivity[^1]. The mental model is closer to Knockout or MobX with modern syntax: a component function runs **once** to set up reactive subscriptions, and individual signal reads inside JSX become targeted DOM updates.

This makes Solid one of the fastest UI libraries in synthetic benchmarks (js-framework-benchmark consistently ranks it near vanilla JS)[^2], and one of the smallest — a non-trivial Solid app runs ~7–10 KB of runtime gzipped. The cost is a mental model that **looks** like React but **behaves** differently in important ways: components don't re-run on update, `setState`-style closure capture doesn't work the same, and destructuring props loses reactivity.

SolidStart is the official meta-framework (Vite-based, file-based routing, server functions). Solid 1.9 (2024) introduced experimental partial hydration; Solid 2.0 is in design with an eye on simplifying the reactivity primitives and converging on `createMemo` / `createEffect` semantics.

## Getting Started

```bash
npx degit solidjs/templates/ts my-app
cd my-app
npm install
npm run dev
```

A minimal Solid component:

```jsx
import { createSignal } from "solid-js"

function Counter() {
  const [count, setCount] = createSignal(0)
  return (
    <button onClick={() => setCount(count() + 1)}>
      count: {count()}
    </button>
  )
}
```

Or with SolidStart:

```bash
npm create solid@latest my-app
```

## Architecture / How It Works

Solid's compiler transforms JSX into imperative DOM-building code with effect wrappers around dynamic expressions[^3]. Static parts of the template are emitted as cloneable HTML templates; dynamic parts become targeted effect functions that update specific text nodes or attributes.

Signals are the core primitive: `createSignal(initial)` returns a `[getter, setter]` tuple. Reading the getter inside an effect or memo subscribes that effect to changes. Writing the setter notifies subscribers synchronously by default.

```jsx
const [count, setCount] = createSignal(0)
const doubled = createMemo(() => count() * 2)
createEffect(() => console.log(count(), doubled()))
```

This is the same primitive that Vue Vapor, Svelte 5 runes, Preact signals, and Angular signals all converged on between 2023–2025. Solid was the first to ship it as the primary API.

**Components run once.** A component function is called during mount to set up the reactive graph. It does not re-run on update — only the specific effects subscribed to changed signals run. This means:
- You don't use `useEffect` semantics; you use `createEffect` for side effects and `onMount` for one-time setup.
- Destructuring props (`function Foo({ x })`) breaks reactivity because the destructure happens once.
- Conditional logic at the component top level runs only once; use `<Show>` / `<For>` components or `Show`/`Switch` JSX helpers for conditional rendering.

## Production Notes

**Reactivity footguns.**
- `<For each={items()}>` is the right pattern for lists; `items().map(...)` re-creates DOM nodes on every change.
- Destructuring props in component signatures breaks reactivity. Use `props.x` and `mergeProps`/`splitProps`.
- `createEffect` callbacks run after the DOM updates; `createRenderEffect` runs before.
- `untrack` is required when you need to read a signal without subscribing.

**Hydration cost.** SolidStart supports both static rendering and SSR with hydration. Hydration in Solid is genuinely cheap because there's no virtual DOM to reconcile — only effect subscriptions to re-establish. Partial hydration (resumability) is experimental as of 1.9.

**Ecosystem.** Much smaller than React or Vue. Solid Primitives (the official utility library) covers most common needs. UI kits include Kobalte, Park UI, and Solid UI (shadcn port). For form validation, `@modular-forms/solid`. For data fetching, `@tanstack/solid-query`.

**SSR + signals.** Signals do not exist on the server in the same way — `createSignal` produces a one-shot value during SSR. Code that relies on signal subscriptions for server-time logic will need restructuring.

**Solid Router.** The official router is `@solidjs/router`. It is competent but has had more breaking changes than equivalents in other ecosystems. Pin versions in production.

## When to Use / When Not

**Use when:**
- You want React-shaped JSX with better default performance.
- Bundle size and runtime perf are first-class requirements.
- You're building a real-time / high-update-frequency UI (charts, dashboards, IDE-like tools).
- You're comfortable teaching the team that components don't re-run.

**Avoid when:**
- You need a large hiring pool — Solid developers are still rare.
- You depend on React-specific libraries with no Solid equivalent (most major React UI kits are React-only).
- Your team will keep writing React patterns (destructured props, `useEffect`-shaped thinking) by reflex.

## Alternatives

- [facebook/react](../facebook/react.md) — same JSX, much larger ecosystem, virtual DOM cost.
- [sveltejs/svelte](../sveltejs/svelte.md) — same compile-time + signals philosophy with SFCs instead of JSX.
- [vuejs/core](../vuejs/core.md) — Proxy-based signals, larger framework.
- [withastro/astro](../withastro/astro.md) — can host Solid islands without committing the whole app.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2018 | Initial Ryan Carniato release. |
| 1.0 | 2021-06 | Stable, JSX + signals[^1]. |
| 1.5 | 2022-09 | Suspense, transitions, concurrent rendering. |
| 1.6 | 2023-01 | SSR streaming, hydration improvements. |
| 1.7 | 2023-05 | Improved error boundaries. |
| 1.8 | 2024-01 | TypeScript improvements, `createAsync`. |
| 1.9 | 2024-09 | Experimental partial hydration. |
| 2.0 | (planned) | Simplified primitives, stable async. |

## References

[^1]: Ryan Carniato, "Introducing the SolidJS UI Library" — 2020. https://dev.to/ryansolid/introducing-the-solidjs-ui-library-4mck
[^2]: krausest, "js-framework-benchmark". https://krausest.github.io/js-framework-benchmark/
[^3]: Ryan Carniato, "Building a Reactive Library from Scratch" — 2021. https://dev.to/ryansolid/building-a-reactive-library-from-scratch-1i0p
