# vercel/swr

> React Hooks for data fetching, built around the stale-while-revalidate cache strategy.

[GitHub repo](https://github.com/vercel/swr) ·
[Official website](https://swr.vercel.app) ·
[License: MIT](https://github.com/vercel/swr/blob/main/LICENSE)

## Overview

SWR is a React Hooks library for reading remote data, created by the team behind Next.js at Vercel and open-sourced in October 2019[^1]. The name comes from `stale-while-revalidate`, an HTTP cache-control strategy from RFC 5861[^2]: return the cached (stale) value immediately, fire the request in the background, then replace it with the fresh result. In practice this means `useSWR(key, fetcher)` renders instantly from cache and reconciles when the network responds.

The defining design choice is that SWR is transport- and protocol-agnostic. It does not fetch anything itself — you supply a `fetcher` function, and SWR owns only the cache, deduplication, and revalidation lifecycle keyed by a string (or serializable) `key`. This keeps the library small (a few KB minzipped) and lets it wrap REST, GraphQL, gRPC, or any async function. The cost of that minimalism is that anything beyond read-and-revalidate — complex mutations, normalized caching, invalidation graphs — is either hand-rolled or belongs in a heavier library.

SWR occupies the "small and declarative" end of the React data-fetching spectrum. Its primary comparison point is TanStack Query, which covers a broader surface (mutations, query invalidation, offline persistence, devtools) at a larger footprint. SWR's development pace has slowed since the 2.0 line, but the repository remains maintained and the library is still widely deployed, particularly in Next.js codebases where it is the house default[^3].

## Getting Started

```bash
npm install swr
```

```jsx
import useSWR from 'swr'

const fetcher = (url) => fetch(url).then((r) => {
  if (!r.ok) throw new Error('request failed')   // fetch does NOT throw on 4xx/5xx
  return r.json()
})

function Profile() {
  const { data, error, isLoading } = useSWR('/api/user', fetcher)

  if (error) return <div>failed to load</div>
  if (isLoading) return <div>loading…</div>
  return <div>hello {data.name}</div>
}
```

The `key` (`/api/user`) is both the cache identity and the argument passed to `fetcher`. Two components mounting the same key share one in-flight request and one cache entry.

## Architecture / How It Works

SWR maintains a single global cache (a `Map`-like store) keyed by the serialized `key`. Array and object keys are serialized deterministically, so `['/api/user', 1]` is a stable identity across renders. Core mechanics:

- **Deduplication.** Requests for the same key within `dedupingInterval` (2s default) collapse into one network call. Mounting the same hook in ten components triggers one fetch.
- **Automatic revalidation.** By default SWR revalidates on window focus (`revalidateOnFocus`), network reconnect (`revalidateOnReconnect`), and optionally on an interval (`refreshInterval`). This is what produces the "always fresh" feel — and the most common surprise (see Production Notes).
- **Conditional / dependent fetching.** Passing `null` (or a function that throws) as the key skips the request, which is how dependent queries and gating are expressed: `useSWR(user ? ['/api/orders', user.id] : null, fetcher)`.
- **Mutation.** `mutate(key, data, options)` updates the cache and optionally revalidates. Bound `mutate` from the hook targets its own key. Optimistic UI uses `optimisticData` + `rollbackOnError`.
- **Configuration.** `<SWRConfig value={{...}}>` sets defaults (fetcher, intervals, `fallback` for SSR hydration) for a subtree.

`useSWRInfinite` handles pagination and "load more" by managing an array of pages. `useSWRMutation` (added in 2.0) provides declarative remote mutations that are not triggered on render. `useSWRSubscription` (2.0) adapts push sources like WebSockets. A middleware API wraps the hook to layer logging, retries, or cache adapters.

The cache is pluggable via a `provider` on `SWRConfig`, but the default provider is in-memory only — there is no built-in persistence.

## Production Notes

- **Focus revalidation causes request storms.** The default `revalidateOnFocus: true` refetches every keyed query each time the tab regains focus. On dashboards with many hooks this can hammer an API. Disable globally via `SWRConfig` or per-hook for expensive endpoints.
- **`fetcher` must throw on error.** The native `fetch` resolves on 404/500; it only rejects on network failure. If your fetcher does not check `res.ok` and throw, SWR never populates `error` and treats an error body as valid data.
- **Keys must be stable and serializable.** Inline object keys that change identity every render (`{ id }` literals differing by reference are fine since SWR serializes, but non-serializable values like functions or class instances are not). Unstable keys defeat deduplication and cause refetch loops.
- **No persistence out of the box.** The cache is cleared on reload. localStorage/IndexedDB persistence requires a custom `provider`; there is no first-party equivalent to TanStack Query's persist plugins.
- **Optimistic updates need explicit rollback.** Without `rollbackOnError`, a failed optimistic mutation leaves the cache showing the optimistic value. Pair `optimisticData` with `rollbackOnError` and a revalidation.
- **SSR/Next.js hydration.** Server-fetched data is passed through `SWRConfig`'s `fallback` (or `fallbackData` per hook) to avoid a loading flash and hydration mismatch. Forgetting this re-fetches on the client on first paint.
- **2.0 migration.** SWR 2.0 changed the cache interface (stricter `Map`-shaped provider), the `mutate` signature, and moved to `useSWRConfig().mutate` for global mutation. Custom cache providers written against 1.x need review.
- **Suspense mode** (`suspense: true`) integrates with React Suspense but historically interacts awkwardly with conditional `null` keys and error boundaries; test error and loading paths before relying on it.

## When to Use / When Not

**Use when:**
- You want read-focused data fetching with minimal API surface and small bundle cost.
- You're in a Next.js/React app and want the default, well-integrated option.
- Your needs are mostly GET-and-revalidate: lists, profiles, dashboards, polling.
- You want to wrap an arbitrary async transport (GraphQL, gRPC, SDK calls) without adopting its client.

**Avoid when:**
- Mutations and cache invalidation are central; TanStack Query's mutation and invalidation model is more complete.
- You need normalized/relational client caching — reach for a GraphQL client with a normalized store.
- You need first-class offline persistence, request cancellation guarantees, or a mature devtools story.
- You're not using React — SWR is React-only.

## Alternatives

- TanStack/query — use instead when mutations, query invalidation graphs, offline persistence, or devtools are central; larger but far broader.
- apollo/apollo-client — use when your backend is GraphQL and you want a normalized cache with field-level updates.
- reduxjs/redux-toolkit (RTK Query) — use when the app already standardizes on Redux and you want data fetching in the same store.
- trpc/trpc — use for end-to-end TypeScript-safe RPC where client and server share types (wraps TanStack Query under the hood).
- tanstack/query (Svelte/Vue/Solid adapters) — use when you need the same model outside React.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2019-10 | Open-sourced at Next.js Conf; `useSWR`, dedup, focus revalidation[^1]. |
| 1.0 | 2021 | Cache API stabilized, smaller core, immutable mode, middleware groundwork. |
| 2.0 | 2022-12 | `useSWRMutation`, `useSWRSubscription`, optimistic UI, mutable cache provider, DevTools[^4]. |

## References

[^1]: Vercel, "Introducing SWR: React Hooks for Remote Data Fetching" — 2019. https://vercel.com/blog/swr
[^2]: HTTP RFC 5861, "HTTP Cache-Control Extensions for Stale Content". https://datatracker.ietf.org/doc/html/rfc5861
[^3]: SWR documentation. https://swr.vercel.app
[^4]: SWR 2.0 documentation and blog. https://swr.vercel.app/blog/swr-v2

## Tags

react, hooks, data-fetching, typescript, cache, stale-while-revalidate, swr, vercel, nextjs, client-state
