# facebook/react

> The library for web and native user interfaces.

[GitHub repo](https://github.com/facebook/react) ·
[Official website](https://react.dev) ·
[License: MIT](https://github.com/facebook/react/blob/main/LICENSE)

## Overview

React is a JavaScript library for building user interfaces, originally released by Facebook in 2013[^1]. It introduced the broader ecosystem to virtual DOM diffing, declarative component composition, and a unidirectional data flow as a default mental model. By 2026 React is no longer the smallest, fastest, or simplest UI library, but it remains the de facto reference implementation that most other libraries position themselves against.

What still makes React distinctive is the size and breadth of its ecosystem rather than the runtime itself: server components, Suspense for data fetching, transitions, and the compiler-based optimization path introduced in React 19[^2] have pushed the library in the direction of a *framework substrate* rather than a view library. Most teams interact with React indirectly through Next.js, Remix, Expo, or React Native rather than the bare `react`/`react-dom` packages.

## Getting Started

```bash
# Most projects today consume React through a framework
npx create-next-app@latest my-app          # Next.js (App Router)
npm create vite@latest my-app -- --template react-ts   # Vite + React + TS
npx create-expo-app my-app                 # React Native (Expo)
```

Bare React in the browser:

```bash
npm install react react-dom
```

```jsx
import { createRoot } from "react-dom/client";

function Counter() {
  const [n, setN] = React.useState(0);
  return <button onClick={() => setN(n + 1)}>count: {n}</button>;
}

createRoot(document.getElementById("root")).render(<Counter />);
```

For server components and the post-19 data model, defer to a framework — the raw `react-server-dom-webpack` / `react-server-dom-turbopack` packages are not intended for direct application use[^3].

## Architecture / How It Works

React's runtime today has three roughly distinct surfaces:

1. **Reconciler (Fiber)** — the scheduling and diffing engine introduced in React 16 (2017)[^4]. Fibers are units of work that allow rendering to be interruptible and prioritized; this is what enables `useTransition`, `useDeferredValue`, and concurrent rendering.
2. **Renderer** — pluggable backends (`react-dom`, `react-native`, `react-three-fiber`, ink, etc.) that translate the reconciler's output into a host environment.
3. **React Server Components (RSC)** — a separate execution boundary introduced as stable in React 19. Server components render on a server (or at build time), serialize their output into a custom wire format, and stream it to a client renderer that may interleave it with client components[^5].

The React Compiler (formerly React Forget), shipped as stable alongside React 19, performs ahead-of-time memoization by analyzing component code at build time. The intent is to eliminate the need for `useMemo` / `useCallback` / `React.memo` in idiomatic component code[^6]. It is opt-in via the `babel-plugin-react-compiler` package and does not yet cover all patterns (notably mutable refs and certain conditional hook calls).

The hooks model (2018) and the concurrent rendering model (2022) were the two largest architectural shifts. RSC and the compiler are the third; they fundamentally change where component code runs and how often it re-executes, and they invalidate a meaningful slice of pre-2024 React tutorials.

## Production Notes

**Memory & re-render profile.** React's per-component overhead is small but non-trivial; in the 1k–10k component range, list virtualization (`react-virtual`, `react-virtuoso`) becomes mandatory rather than optional. The compiler reduces unnecessary re-renders but does not eliminate the cost of mounting; large lists still need windowing.

**Hydration cost.** RSC reduces JavaScript shipped to the client but does not eliminate hydration cost for client components. Initial Time to Interactive on resource-constrained devices is still dominated by hydration in most non-trivial apps[^7]. If your audience is mostly low-end Android, measure before assuming RSC is a free win.

**State management churn.** Redux, MobX, Zustand, Jotai, Recoil, Valtio, and React's own `useReducer` + Context are all in production use. React Query / TanStack Query has become the closest thing to a default for server state. RSC blurs the line further by moving some "state" onto the server entirely.

**Migration pain.**
- React 18 → 19 is mostly compatible but removes legacy APIs (`ReactDOM.render`, `findDOMNode`, string refs, legacy context). Codemods are available[^8].
- Class components still work but are no longer mentioned in new docs.
- The `Suspense` boundary semantics changed between 17 and 18 — error handling order is different.
- `act()` from `react-dom/test-utils` was moved to `react`; older test suites need updates.

**Concurrent rendering footguns.** Effects can fire in unexpected orders under concurrent rendering, especially when combined with external mutable stores. The `useSyncExternalStore` hook exists specifically to make external stores concurrent-safe; any store integration written before 2022 should be audited.

## When to Use / When Not

**Use when:**
- You need the widest possible hiring pool and ecosystem.
- You're building a product that will outlive any single framework's hype cycle and want a substrate that has survived multiple of them.
- You need React Native or a shared component model across web + native.

**Avoid when:**
- You're chasing minimum bundle size: Svelte, Solid, or Preact all produce smaller bundles for equivalent apps.
- You want fine-grained reactivity without a compiler: Solid and Vue 3 (with `<script setup>`) are simpler mental models.
- You're building a static content site: Astro with islands is generally a better fit than React + RSC.
- You need first-class signals / fine-grained reactivity: React's compiler approximates this but is not a replacement.

## Alternatives

- [vuejs/core](../vuejs/core.md) — option/composition API, fine-grained reactivity, smaller default bundle.
- [sveltejs/svelte](../sveltejs/svelte.md) — compile-away runtime, smallest bundles in this class.
- [solidjs/solid](../solidjs/solid.md) — signals-based, JSX-compatible, much smaller runtime.
- [withastro/astro](../withastro/astro.md) — islands architecture; can host React components without shipping React to every page.
- Preact (not in this wiki yet) — drop-in 3kb React-compatible alternative for legacy / size-critical work.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.3 | 2013-05 | Initial open source release at JSConf US[^1]. |
| 0.14 | 2015-10 | `react-dom` split from `react`. |
| 15.0 | 2016-04 | SVG support, refactor of DOM diffing. |
| 16.0 | 2017-09 | Fiber reconciler, fragments, error boundaries[^4]. |
| 16.8 | 2019-02 | Hooks (`useState`, `useEffect`, …)[^9]. |
| 17.0 | 2020-10 | No new features; gradual upgrade target. |
| 18.0 | 2022-03 | Concurrent rendering, automatic batching, `useTransition`, streaming SSR[^10]. |
| 19.0 | 2024-12 | Stable Server Components, Actions, `use()` hook, React Compiler (opt-in)[^2]. |

## References

[^1]: Jordan Walke et al., "Introducing React" — Facebook Engineering, 2013-06-05. https://legacy.reactjs.org/blog/2013/06/05/why-react.html
[^2]: React Team, "React 19" — 2024-12-05. https://react.dev/blog/2024/12/05/react-19
[^3]: React docs, "Server Components". https://react.dev/reference/rsc/server-components
[^4]: Andrew Clark, "React Fiber Architecture". https://github.com/acdlite/react-fiber-architecture
[^5]: React Team, "Introducing Zero-Bundle-Size React Server Components" — 2020-12-21. https://legacy.reactjs.org/blog/2020/12/21/data-fetching-with-react-server-components.html
[^6]: React docs, "React Compiler". https://react.dev/learn/react-compiler
[^7]: Addy Osmani, "The cost of JavaScript". https://v8.dev/blog/cost-of-javascript-2019 (still broadly applicable to hydration)
[^8]: React docs, "React 19 Upgrade Guide". https://react.dev/blog/2024/04/25/react-19-upgrade-guide
[^9]: React Team, "Introducing Hooks" — 2019-02-06. https://legacy.reactjs.org/blog/2019/02/06/react-v16.8.0.html
[^10]: React Team, "React 18" — 2022-03-29. https://react.dev/blog/2022/03/29/react-v18
