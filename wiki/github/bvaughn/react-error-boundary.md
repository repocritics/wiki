# bvaughn/react-error-boundary

> A reusable React error boundary component — the de facto replacement for hand-rolled `componentDidCatch` classes.

[GitHub repo](https://github.com/bvaughn/react-error-boundary) ·
[Official website](https://react-error-boundary-lib.vercel.app) ·
[License: MIT](https://github.com/bvaughn/react-error-boundary/blob/main/LICENSE)

## Overview

`react-error-boundary` is a small library that wraps React's error-boundary primitive in a configurable, reusable component plus a hook. React only supports error boundaries as class components — you must implement `static getDerivedStateFromError` and `componentDidCatch` by hand, and there is no function-component or hook API for it[^1]. This package supplies that class once, exposes three ways to render a fallback, and adds the reset/retry plumbing most apps end up re-inventing. It is maintained by Brian Vaughn, a former React core-team member (React DevTools, react-window)[^2], which is a large part of why it became the community default rather than one of several competing wrappers.

The library is deliberately tiny: a single `ErrorBoundary` component, a `useErrorBoundary` hook, a `withErrorBoundary` HOC, and a few helpers such as `getErrorMessage`. It has no runtime dependencies beyond React (declared as a peer) and works across every React renderer — React DOM, React Native, and others — because it only touches the renderer-agnostic error-boundary contract[^1].

The defining tension is scope. Error boundaries catch errors thrown *while rendering*; they do not catch errors in event handlers, `setTimeout`/promise callbacks, server-side rendering, or errors thrown inside the boundary itself. Newcomers frequently expect a boundary to be a global `try/catch` for their component tree and are surprised when a click-handler exception sails straight past it. This library does not — and cannot — change that; it gives you `useErrorBoundary().showBoundary(error)` to forward such errors manually, but the boundary/render-phase distinction is inherited from React.

## Getting Started

```sh
npm install react-error-boundary
# or: pnpm add react-error-boundary / yarn add react-error-boundary
```

```tsx
"use client"; // required in React Server Component frameworks — this is a client component

import { ErrorBoundary } from "react-error-boundary";

function Fallback({ error, resetErrorBoundary }) {
  return (
    <div role="alert">
      <p>Something went wrong:</p>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

export default function App() {
  return (
    <ErrorBoundary
      FallbackComponent={Fallback}
      onError={(error, info) => reportToService(error, info.componentStack)}
      onReset={() => {/* clear the state that caused the error */}}
    >
      <Widget />
    </ErrorBoundary>
  );
}
```

For errors that boundaries do not catch (event handlers, async), forward them manually:

```tsx
import { useErrorBoundary } from "react-error-boundary";

function Widget() {
  const { showBoundary } = useErrorBoundary();
  async function onClick() {
    try { await risky(); } catch (err) { showBoundary(err); }
  }
  return <button onClick={onClick}>Do it</button>;
}
```

## Architecture / How It Works

Under the hood `ErrorBoundary` is still a React class component — the library does not escape React's requirement, it encapsulates it. `getDerivedStateFromError` stores the thrown value in state, `componentDidCatch` invokes the `onError` callback, and on the next render the component branches to a fallback instead of `children`.

It offers three mutually exclusive ways to render that fallback, which is the main source of API confusion:

- **`fallback`** — a static React element rendered as-is (no access to the error).
- **`FallbackComponent`** — a component that receives `{ error, resetErrorBoundary }` as props.
- **`fallbackRender`** — a render-prop function receiving the same object inline.

Reset semantics are the non-obvious part. `resetErrorBoundary()` clears the internal error state and re-renders `children` — but if whatever caused the error (a bad prop, stale state) is still present, the render immediately throws again, producing an error loop. The intended pattern is to pair reset with `onReset` (to clear the offending state) and/or `resetKeys` — an array the boundary shallow-compares between renders; when any element changes identity, the boundary auto-resets. This lets you tie recovery to specific state (`resetKeys={[userId]}`) rather than a manual button.

The `useErrorBoundary` hook (named `useErrorHandler` before v4) closes the render-phase gap: it returns `showBoundary(error)` which schedules a state update that re-throws during render, so the *nearest* boundary catches it. In React 19 this overlaps with `useTransition` — errors thrown from a `startTransition` async action also propagate to the nearest boundary — so the hook is less necessary but still the explicit path[^3].

## Production Notes

**The catch scope is the number-one footgun.** Boundaries catch render-phase errors only. Event-handler and async errors must be routed through `showBoundary` or a `useTransition` action, or they escape to `window.onerror`. Auditing "did we actually wrap the async paths?" is a real review item, not a formality.

**`resetKeys` must contain stable values.** Passing a freshly-constructed array or object literal as a reset key on every render (`resetKeys={[{ id }]}`) causes the boundary to reset on every render, silently defeating itself. Use primitives or memoized references.

**SSR / RSC.** Error boundaries do not catch errors thrown during server-side rendering. In Next.js App Router, render errors on the server are handled by the framework's `error.tsx` convention, not by this component — dropping `ErrorBoundary` into a Server Component does nothing until you mark it `"use client"`, and even then it only guards client renders.

**v6 is ESM-only.** As of version 6, the package ships ES Modules only; projects on bundlers or runtimes without ESM support must pin to version 5, which the README calls out explicitly[^4]. This is the most common upgrade blocker for older toolchains.

**`@types/react` mismatch.** A "`ErrorBoundary` cannot be used as a JSX component" TypeScript error is almost always a version skew between `react` and `@types/react`; the fix is an `overrides`/`resolutions` pin so both match, not a change to your code[^4].

**Pairing with data libraries.** For query libraries, TanStack Query exposes `QueryErrorResetBoundary` designed to be composed with this component so that resetting the boundary also retries the failed query — the two are meant to be used together, not as substitutes.

**Health.** Actively maintained: ~7,970 stars, 228 forks, last pushed 2026-07, and an open-issue count at zero at time of writing — the author keeps the tracker aggressively triaged rather than it signaling low usage. The API has been stable since the v4 redesign (2023); v5 and v6 were mostly packaging changes.

## When to Use / When Not

**Use when:**
- You want error boundaries without hand-writing class components in every app.
- You need per-section fallback UI with a retry/reset story (`resetKeys`, `onReset`).
- You're on React Native or a non-DOM renderer and want a portable boundary.
- You want a single, well-audited dependency instead of copy-pasted boundary classes.

**Avoid when:**
- Your framework already provides route-level boundaries (Next.js `error.tsx`, Remix/React Router route `ErrorBoundary`) and that granularity is enough.
- You need to catch event-handler or async errors *without* explicit forwarding — no boundary library can do that.
- You want zero dependencies and only need one boundary — a ~15-line class component is a legitimate alternative.
- You're on a non-ESM toolchain and cannot upgrade past v5.

## Alternatives

- getsentry/sentry-javascript — `@sentry/react`'s `ErrorBoundary` bundles the boundary with automatic error reporting; use it when you already ship Sentry and want capture built in.
- vercel/next.js — App Router `error.tsx` gives route-segment boundaries for free; use when a whole-route fallback suffices and you don't need in-tree granularity.
- remix-run/react-router — framework-level route `ErrorBoundary` exports; use when your errors map naturally to route boundaries.
- TanStack/query — not a replacement but the standard companion (`QueryErrorResetBoundary`) when boundary resets should retry queries.
- A hand-rolled class component — use when you need exactly one boundary and want zero dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2017 | Initial release; basic reusable boundary component[^2]. |
| 3.x | ~2020–2022 | `useErrorHandler` hook, `FallbackComponent` API[^3]. |
| 4.0.0 | 2023 | API redesign: `useErrorHandler` → `useErrorBoundary`, added `fallbackRender`, `fallback`, `resetKeys`. |
| 5.0.0 | 2024-12-21 | Last version supporting non-ESM environments[^4]. |
| 6.0.0 | 2025-05-03 | ESM-only build. |
| 6.1.0 | 2026-01-17 | Feature additions on the 6.x line. |
| 6.1.2 | 2026-05-25 | Latest patch at time of writing. |

## References

[^1]: React docs, "Component — Catching rendering errors with an error boundary." https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary
[^2]: Repository metadata and history, bvaughn/react-error-boundary (created 2017-07-18). https://github.com/bvaughn/react-error-boundary
[^3]: Kent C. Dodds, "Use react-error-boundary to handle errors in React" (documents the v3-era API). https://kentcdodds.com/blog/use-react-error-boundary-to-handle-errors-in-react
[^4]: react-error-boundary README, ESM note and `@types/react` FAQ. https://github.com/bvaughn/react-error-boundary#readme

## Tags

react, error-boundary, error-handling, typescript, react-native, frontend, component-library, resilience, javascript
