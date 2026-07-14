# preactjs/preact

> A ~4kB alternative to React with the same modern API — the same mental model, a fraction of the runtime.

[GitHub repo](https://github.com/preactjs/preact) ·
[Official website](https://preactjs.com) ·
[License: MIT](https://github.com/preactjs/preact/blob/main/LICENSE)

## Overview

Preact is a virtual-DOM UI library created by Jason Miller (developit) in 2015[^1]. Its
premise is narrow and durable: reimplement the parts of React's component model that
applications actually use — function/class components, hooks, JSX, context — in a core
that ships in a few kilobytes gzipped rather than React's tens of kilobytes. The API is
deliberately React-shaped so that knowledge, JSX, and much tooling transfer directly.

The defining tension is compatibility versus size. Preact's own core is intentionally
small and does not aim for byte-for-byte React behavior. Full React interoperability —
so that packages written against `react`/`react-dom` run unmodified — lives in a separate
`preact/compat` alias layer that you opt into via bundler aliasing. This split is the
source of most of Preact's real-world friction: things work until a dependency depends on
a React internal or behavioral detail the thin core does not replicate.

Preact is not a framework. It has no router, no data layer, and no build tooling of its
own. It is the render layer, and the surrounding ecosystem (preact-router, or Preact's
own meta-framework work, plus the broader React ecosystem via compat) fills the rest. Its
largest single deployment surface is embedded and size-sensitive contexts — widgets,
embeds, PWAs, and the many sites that use it under the hood without their users knowing.

## Getting Started

```bash
npm install preact
# optional: run React-ecosystem packages against Preact
npm install preact/compat   # actually an alias target, configured in your bundler
```

```jsx
// Modern JSX transform + hooks. No `h` pragma needed with automatic runtime.
import { render } from 'preact';
import { useState } from 'preact/hooks';

function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>Count: {n}</button>;
}

render(<Counter />, document.getElementById('app'));
```

Hooks are not in the core bundle — they live in `preact/hooks` and are imported
separately, which keeps the base render path small for code that does not use them.

## Architecture / How It Works

The core exports a `createElement`/`h` factory, a `render(vnode, parent)` entry point, and
`Component`. Rendering builds a tree of virtual nodes (VNodes) and diffs each against the
previously committed tree, mutating the real DOM with as few operations as possible. Since
Preact X (v10, 2019) the diff is a rewrite that added fragments, `componentDidCatch` error
boundaries, a full context API, and hooks support[^2].

Several design choices distinguish it from React internally:

- **Native DOM events, not a synthetic system.** Preact attaches real listeners and leans
  on the browser's own event model. This removes React's synthetic-event abstraction and
  its bundle cost, but means event names and semantics follow the DOM (`onInput` fires as
  you type; there is no synthetic `onChange` normalization). `preact/compat` patches some
  of these differences back for React parity.
- **Props map closer to HTML attributes.** Preact accepts `class` as well as `className`,
  and passes many props through to the DOM more directly than React does.
- **Hooks as an add-on.** `preact/hooks` implements `useState`, `useEffect`, `useMemo`,
  etc. against the core's component internals rather than being baked into the renderer.
- **`preact/compat`** re-exports a `react`/`react-dom`-shaped surface (including
  `forwardRef`, `memo`, `Children`, `createPortal`) implemented on top of the thin core.
  You wire it up by aliasing `react` and `react-dom` to `preact/compat` in your bundler.

Signals (`@preact/signals`) are a separate, optional reactivity model — fine-grained
observable state that updates the DOM without re-running whole components[^3]. It is not
part of core and represents a different rendering philosophy than the top-down VDOM diff.

The coupling story: your app couples to the React *shape*, not to React. Migrating an
existing React app is usually an aliasing exercise plus fixing the handful of places that
touch behavior compat does not cover. Migrating *away* from Preact is similarly bounded.

## Production Notes

**`compat` is where surprises live.** The core is small precisely because it does not
replicate every React behavior; `preact/compat` covers the common surface but not all of
it. Libraries that reach into React internals, rely on synthetic event pooling/ordering,
or depend on exact reconciliation timing can misbehave in ways that are hard to attribute.
Budget time to test third-party React components rather than assuming drop-in parity.

**Bundler aliasing is a required, easy-to-miss step.** React-ecosystem interop depends on
aliasing `react`→`preact/compat` (and `react-dom`, `react/jsx-runtime`) in webpack/Vite/
esbuild config. Miss it and you silently pull in real React alongside Preact, defeating
the size win and sometimes running two renderers at once.

**Event-model differences bite during migration.** Code written for React's synthetic
events — particularly `onChange` on inputs, and assumptions about event bubbling/pooling —
may behave differently. These are the most common per-component migration fixes.

**TypeScript.** Preact ships its own types; with `compat` you generally point JSX types at
Preact's. Mixed setups (some deps expecting `@types/react`) can produce type conflicts that
require `paths`/alias configuration in `tsconfig` to resolve.

**Version boundaries.** Preact X (v10) has been the stable line for years and is where the
current ecosystem sits. The `main` branch tracks an upcoming major (v11); v10 patches live
on the `v10.x` branch[^4]. Pin to v10 for production today unless you are deliberately
tracking prereleases, and read migration notes before adopting the next major.

**Size claims are for the core only.** The advertised ~4kB is preact core gzipped; real
apps add `preact/hooks`, `compat`, signals, router, and your own code. The library still
lands far smaller than an equivalent React build, but the headline number is not your
bundle size.

## When to Use / When Not

**Use when:**
- Bundle size is a hard constraint: embeds, widgets, ads, PWAs, low-bandwidth markets.
- You want the React programming model and JSX without React's runtime footprint.
- You are building fresh and can stay on the thin core plus `preact/hooks`.
- You want an optional fine-grained reactivity path (signals) alongside VDOM.

**Avoid when:**
- You depend heavily on React-only libraries that touch internals or synthetic events —
  compat may not fully cover them, and debugging the gaps costs more than the size saved.
- You want a batteries-included framework: Preact is the render layer only.
- You need guaranteed byte-for-byte React behavior for a large existing codebase; audit
  before committing to a migration.

## Alternatives

- facebook/react — the reference implementation; choose it when ecosystem parity and
  React-internal-dependent libraries matter more than bundle size.
- solidjs/solid — signals-first, compile-time reactivity, no VDOM; use when you want
  fine-grained updates as the default model rather than an add-on.
- sveltejs/svelte — compiler that emits minimal runtime; use when you can adopt a
  non-JSX component syntax for the smallest output.
- vuejs/core — comparable size-to-features balance with its own reactivity system; use
  when you prefer Vue's template model over React-shaped JSX.
- developit/htm — JSX-less tagged-template syntax from the same author; pair with Preact
  when you want no build step at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-09 | First release by Jason Miller; tiny VDOM core[^1]. |
| 8.x | 2017 | Widely adopted small-core line preceding the X rewrite. |
| X (10.0) | 2019-05 | Full rewrite: fragments, hooks, error boundaries, context API[^2]. |
| signals | 2022-09 | `@preact/signals` fine-grained reactivity released separately[^3]. |
| 10.x | ongoing | Long-lived stable line; patches on the `v10.x` branch[^4]. |
| 11 (upcoming) | — | Next major in development on `main`[^4]. |

## References

[^1]: Preact repository, created 2015-09-11; author Jason Miller (developit). https://github.com/preactjs/preact
[^2]: "Preact X, A Story of a Rewrite" — Preact blog on the v10 line. https://preactjs.com/blog/preact-x/
[^3]: "Introducing Signals" — Preact blog, 2022. https://preactjs.com/blog/introducing-signals/
[^4]: Preact README branch note: `main` tracks the upcoming release; v10 patches on `v10.x`. https://github.com/preactjs/preact/tree/v10.x

## Tags

javascript, ui-library, virtual-dom, react-alternative, jsx, frontend, hooks, components, lightweight, spa
