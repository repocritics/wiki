# TanStack/query

TanStack Query (formerly React Query) — server-state management library for web apps. The canonical "fetch + cache + refetch" library.

## What it is

A TypeScript library that manages server state (data fetched from APIs) separately from client state. Provides caching, deduplication, background refetching, pagination, infinite queries, mutations with optimistic updates, and devtools. Framework-agnostic core with adapters for React, Vue, Svelte, Solid, Angular.

## Key features

- Caching, deduplication, stale-while-revalidate semantics.
- Background refetching on window focus / network reconnect / interval.
- Pagination + infinite scrolling helpers.
- Mutations with optimistic updates + rollback.
- Persistence (LocalStorage, IndexedDB) for offline.
- Devtools for cache inspection.
- Multi-framework: React, Vue, Svelte, Solid, Angular.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Framework adapters via separate packages (`@tanstack/react-query`, etc.).

## When to reach for it

- You're fetching server data and want caching + refetch logic out of the box.
- You're replacing ad-hoc useEffect-based data fetching.
- You want the canonical "server state" library.

## When *not* to reach for it

- All your state is client-only — use Zustand / Jotai / Redux.
- You're at the smallest scale where plain `fetch` + useState is enough.

## Maturity signal

Actively maintained under Tanner Linsley + the TanStack team. 5+ years.

## Alternatives

- SWR — comparable lighter alternative.
- Apollo Client — GraphQL-specific.
- RTK Query — for Redux-flavored.

## Tags

typescript, react, vue, svelte, library, server-state, mit-license, caching, data-fetching
