# alibaba/hooks

> A curated React Hooks library, best known for `useRequest` — a plugin-based async data-fetching hook that predates and parallels react-query.

[GitHub repo](https://github.com/alibaba/hooks) ·
[Official website](https://ahooks.js.org/) ·
[License: MIT](https://github.com/alibaba/hooks/blob/master/LICENSE)

## Overview

ahooks (the npm package name; the repo is `alibaba/hooks`) is a collection of ~70 React Hooks maintained inside Alibaba, published under the MIT license. It began life as `@umijs/hooks` in the UmiJS ecosystem in 2019, was renamed to ahooks, and was rewritten as 3.0 in 2021 with a TypeScript-first codebase and a plugin-based `useRequest`[^1]. As of this writing it sits at roughly 15k stars and 2.8k forks, with the last push in May 2026 — actively maintained, but at a mature, low-churn cadence rather than under heavy new-feature development.

The library is two things stacked together. The larger part is a broad utility set — state helpers (`useBoolean`, `useSetState`, `useLocalStorageState`), effect variants (`useUpdateEffect`, `useDebounceEffect`, `useInterval`), and DOM hooks (`useClickAway`, `useSize`, `useKeyPress`) — that overlaps heavily with streamich/react-use. The smaller but defining part is `useRequest`, an async-state manager with polling, debounce/throttle, SWR-style caching, retry, and dependency-driven refetch, built on an internal plugin lifecycle. Most teams adopt ahooks for `useRequest` and get the rest as a bundle.

The defining design decision — and the source of most of its footguns — is that ahooks aggressively wraps user callbacks in refs so they always see the latest closure. This "special treatment for functions" (the README's phrasing) eliminates a whole class of stale-closure bugs, at the cost of breaking React's referential-identity model in ways that interact awkwardly with the exhaustive-deps lint and with concurrent rendering.

## Getting Started

```bash
npm install --save ahooks
# or: yarn add ahooks / pnpm add ahooks / bun add ahooks
```

```tsx
import { useRequest } from "ahooks";

function UserProfile({ id }: { id: string }) {
  const { data, loading, error, refresh } = useRequest(
    () => fetch(`/api/user/${id}`).then((r) => r.json()),
    {
      refreshDeps: [id],        // refetch when id changes
      pollingInterval: 30_000,  // poll every 30s
      cacheKey: `user-${id}`,   // SWR-style cache + revalidate
      debounceWait: 300,
    }
  );

  if (loading && !data) return <span>Loading…</span>;
  if (error) return <span>{error.message}</span>;
  return <button onClick={refresh}>{data.name}</button>;
}
```

Tree-shaking works via named ESM imports; you pay bundle cost only for the hooks you import, though a few (`useRequest`, deep-compare effects) pull in lodash utilities.

## Architecture / How It Works

**Monorepo.** The repo is a pnpm workspace built with UmiJS's `father` bundler. The main artifact is the `ahooks` package; a few hooks are split into scoped packages — notably `use-url-state`, which is published separately as `@ahooksjs/use-url-state` and must be installed under that name[^2]. This split is a recurring source of "module not found" confusion after upgrades.

**Hook categories.** The library organizes hooks into groups documented on the site: `useRequest`, Scene (`useAntdTable`, `useInfiniteScroll`, `usePagination`, `useVirtualList`, `useDynamicList`), LifeCycle (`useMount`, `useUnmount`), State, Effect, Dom, Advanced, and Dev (`useWhyDidYouUpdate`, `useTrackedEffect`). The Scene hooks are opinionated and lean toward the Ant Design / Alibaba mid-office (中台) UI patterns they were extracted from.

**`useRequest` plugin engine.** The hook is a thin wrapper around an internal `Fetch` class exposing a fixed lifecycle — `onBefore`, `onRequest`, `onSuccess`, `onError`, `onFinally`, `onCancel`, `onMutate`. Each feature is a separate plugin (`usePollingPlugin`, `useCachePlugin`, `useDebouncePlugin`, `useRetryPlugin`, `useLoadingDelayPlugin`, `useAutoRunPlugin`, etc.) that hooks into those points. This makes the feature set composable and individually testable, but it also means behavior emerges from plugin interaction order, which is not obvious from the options object alone.

**The ref-wrapping core.** `useMemoizedFn` (v2's `usePersistFn`) returns a function whose identity is stable forever but whose body always calls the latest closure via a ref. `useLatest`, `useCreation`, and `useReactive` build on the same idea. `useReactive` goes furthest: it returns a Proxy-backed mutable object so you can write `state.count++` Vue-style. This is deliberately un-React-like and is the hook most likely to misbehave under `StrictMode` and concurrent features.

## Production Notes

- **`useMemoizedFn` hides dependency bugs.** Because the returned function is referentially stable but semantically changing, `react-hooks/exhaustive-deps` will happily let you omit real dependencies. This is the intended ergonomics, but it means a genuinely-missing dependency and a deliberately-stabilized callback look identical in review. Prefer it consciously, not by default.

- **`useReactive` and concurrent mode.** Mutation-based Proxy state does not participate in React's snapshotting model; it can tear or drop updates under concurrent rendering and `useSyncExternalStore`-style expectations. Treat it as a convenience for local, non-critical state, not app-wide state.

- **`useRequest` cache is in-process and global.** `cacheKey` lives in a module-level map for the page's lifetime; it is not persisted and not namespaced per component tree. Two unrelated components sharing a `cacheKey` string will share (and revalidate) the same data — usually intended, occasionally a surprise. There is no built-in cross-tab or storage persistence like react-query's persisters.

- **StrictMode double-invoke.** Under React 18 dev `StrictMode`, auto-running `useRequest` fires its request twice on mount; polling and cancellation logic assume single mount. This is a dev-only artifact but noisy in logs and can confuse manual-mode assumptions.

- **Deep-compare hooks are lodash `isEqual` every render.** `useDeepCompareEffect` and friends run a full structural comparison on each render. On large or frequently-changing objects this is a real cost; a stable key or normalized primitive dep is cheaper.

- **v2 → v3 was a breaking rename.** The 3.0 rewrite renamed many hooks (`usePersistFn` → `useMemoizedFn`, the old `useAsync` folded into `useRequest` with a different API) and changed defaults. Mixed-version monorepos and stale tutorials referencing v2 names are common upgrade friction; consult the v3 migration guide[^1].

- **Bilingual, China-first community.** Docs are English + 简体中文, but a large share of issues, discussions, and design rationale is in Chinese. English-only maintainers should expect to translate when digging into edge cases.

## When to Use / When Not

**Use when:**
- You want `useRequest`'s batteries-included async ergonomics (polling, debounce, retry, cache) without wiring react-query.
- You're already in an Ant Design / Alibaba-style admin UI and want the Scene hooks (`useAntdTable`, `usePagination`, `useInfiniteScroll`).
- You want one dependency covering both a utility-hook grab-bag and data fetching.

**Avoid when:**
- Server-state caching and invalidation are your core problem — TanStack Query offers stronger cache semantics, devtools, and persistence.
- You want minimal, tree-shaken utility hooks only — smaller curated sets (react-hookz/web, uidotdev/usehooks) carry less surface area and fewer opinions.
- You depend on strict concurrent-mode correctness — the ref-wrapping and `useReactive` patterns cut against React's rendering model.

## Alternatives

- TanStack/query — use instead when server-state caching, invalidation, and devtools are the point; more robust than `useRequest`'s in-memory cache.
- vercel/swr — use instead when you want a smaller, focused stale-while-revalidate fetcher without the rest of the hook library.
- streamich/react-use — use instead when you want the broad utility-hook grab-bag but not `useRequest`; larger and less curated.
- react-hookz/web — use instead when you want a modern, tree-shaken, TypeScript-first utility set with fewer opinions.
- uidotdev/usehooks — use instead when you want a tiny, copy-friendly collection of common hooks rather than a framework-adjacent dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| @umijs/hooks / umi-hooks | 2019 | Original incarnation inside the UmiJS ecosystem[^3]. |
| ahooks 2.x | 2020 | Renamed to ahooks; basic + advanced hooks, `usePersistFn` ref pattern. |
| 3.0 | 2021 | TypeScript rewrite, plugin-based `useRequest`, hook renames[^1]. |
| current | 2026-05 | Actively maintained; last push 2026-05-20. |

## References

[^1]: ahooks documentation and v3 guide. https://ahooks.js.org/
[^2]: README notice: `use-url-state` is published as `@ahooksjs/use-url-state`. https://github.com/alibaba/hooks/blob/master/README.md
[^3]: Repo topics include `umi-hooks` and `ahooks`, reflecting the UmiJS origin. https://github.com/alibaba/hooks

## Tags

react, react-hooks, typescript, hooks-library, data-fetching, use-request, frontend, state-management, alibaba, ssr
