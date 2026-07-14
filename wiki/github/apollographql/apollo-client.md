# apollographql/apollo-client

> A GraphQL client for JavaScript whose defining feature — and defining liability — is a normalized, reactive client-side cache.

[GitHub repo](https://github.com/apollographql/apollo-client) ·
[Official website](https://apollographql.com/client) ·
[License: MIT](https://github.com/apollographql/apollo-client/blob/main/LICENSE)

## Overview

Apollo Client is the most widely deployed GraphQL client in the JavaScript ecosystem, first released in 2016 and rewritten around React hooks and a normalized cache for the 3.0 line in 2020[^1]. It is framework-agnostic at the core but overwhelmingly used with React; official React bindings ship in the box, while Vue, Angular, and Svelte integrations are community- or partner-maintained. Apollo the company sells GraphOS, Router, and Federation tooling; the client is the free, MIT-licensed on-device piece of that stack.

The product it actually sells is not "fetch GraphQL" — that is a few lines of `fetch`. It is the **InMemoryCache**: a normalized store that splits query results into individual objects keyed by `__typename` plus `id`, so that a mutation or a second query updating one object automatically re-renders every component reading that object. This is genuinely valuable for interconnected UIs (a user's name changes in one place and updates everywhere), and it is the single largest source of confusion, bugs, and bundle weight in the library. Most complaints about Apollo Client are, at root, complaints about cache normalization behaving in ways the developer did not model.

The central tension: if your app benefits from a normalized graph cache, few tools do it better; if it does not, you are paying a large tax (bundle size, mental model, merge functions) for a feature you could replace with a document cache in a fraction of the code.

## Getting Started

```sh
npm install @apollo/client graphql
```

```tsx
import {
  ApolloClient,
  InMemoryCache,
  ApolloProvider,
  gql,
  useQuery,
} from "@apollo/client";

const client = new ApolloClient({
  uri: "https://api.example.com/graphql",
  cache: new InMemoryCache(),
});

const GET_DOGS = gql`
  query GetDogs {
    dogs {
      id
      breed
    }
  }
`;

function Dogs() {
  const { data, loading, error } = useQuery(GET_DOGS);
  if (loading) return <p>Loading…</p>;
  if (error) return <p>{error.message}</p>;
  return <ul>{data.dogs.map(d => <li key={d.id}>{d.breed}</li>)}</ul>;
}

export default function App() {
  return (
    <ApolloProvider client={client}>
      <Dogs />
    </ApolloProvider>
  );
}
```

Note the explicit `id` in the selection set: without a normalizable key, the object is cached under its parent query and loses the cross-query update behavior that is the reason to use Apollo at all.

## Architecture / How It Works

Three pieces compose every Apollo Client instance:

1. **`ApolloLink` chain** — a middleware pipeline for the network. Links are composable; a chain typically ends in a *terminating link* (`HttpLink`, `BatchHttpLink`, or a WebSocket/`graphql-ws` link for subscriptions) preceded by non-terminating links (`setContext` for auth headers, `RetryLink`, `ErrorLink`). This is where auth, batching, retries, and error interception live.
2. **`InMemoryCache`** — the normalized store. Query results are flattened into records keyed by cache ID (`Type:id` by default, overridable via `keyFields` in type policies). Reads are answered from the cache when possible; the *fetch policy* (`cache-first` default, `cache-and-network`, `network-only`, `no-cache`, `cache-only`) governs the cache/network tradeoff per query.
3. **Reactive query layer** — `useQuery` and friends subscribe a component to the specific cache entries its query touches. When those entries change (from any mutation, refetch, or manual `cache.writeQuery`), the component re-renders. This is the payoff of normalization.

**Type policies and field policies** are the configuration surface for the cache: `keyFields` change how an object is identified, `merge` functions define how incoming field values combine with existing ones (mandatory for pagination — see `relayStylePagination` / `offsetLimitPagination` helpers), and `keyArgs` control how arguments partition a field's cache storage. Getting pagination right in Apollo means writing correct `merge` functions, not calling a paginate API.

**Local state** is handled by the same cache: reactive variables (`makeVar`), the `@client` directive, and field policies replaced the old `apollo-link-state` / Redux integration. **React 18/19 support** added `useSuspenseQuery`, `useBackgroundQuery`, `useReadQuery`, and `useFragment` for Suspense-based data loading and fragment-scoped subscriptions.

## Production Notes

**Cache normalization is the footgun.** The recurring failure mode is objects returned without an `id` (or without a configured `keyFields`), which the cache cannot normalize — you get stale reads, duplicated data, and `Cache data may be lost…` / `Missing field` console warnings. Any object you expect to update across queries must carry a stable identifier in every selection set that fetches it.

**Pagination requires manual merge functions.** `fetchMore` alone does not merge pages; you must define a field policy `merge` (the built-in `relayStylePagination()` / `offsetLimitPagination()` helpers cover common cases). The older `fetchMore({ updateQuery })` pattern is deprecated in favor of field policies.

**Unbounded cache growth.** The normalized cache holds everything ever fetched until explicitly released. Long-lived SPAs need `cache.evict()` + `cache.gc()` discipline, or the cache becomes a memory leak. There is no automatic TTL/LRU eviction.

**Bundle size.** Historically one of the heaviest data-fetching dependencies in the React ecosystem (tens of KB min+gzip once cache and links are included). Tree-shaking helps but the normalized cache is not optional if you use it. Teams that only need document caching frequently find the weight unjustified.

**SSR and React Server Components are awkward.** Classic SSR uses `getDataFromTree`. Next.js App Router / RSC support historically required a separate integration package (Apollo's Next.js integration, previously `@apollo/experimental-nextjs-app-support`), because a client built around a browser cache does not map cleanly onto server components[^2]. Confirm the current integration package before wiring it into an App Router project.

**Error handling has two axes.** `error` combines `graphQLErrors` (returned in the GraphQL response body, HTTP 200) and `networkError` (transport failure). The `errorPolicy` option (`none` default, `ignore`, `all`) decides whether partial data alongside errors is surfaced or discarded — the default silently drops partial data, which surprises teams expecting `all`.

**TypeScript.** Types are strong but hand-written generics on hooks are error-prone; production setups pair Apollo with GraphQL codegen (`@graphql-codegen` or Apollo's typed-document-node output) so operations carry their result and variable types automatically.

**Upgrade pain.** The 2.x → 3.0 jump was a rewrite (hooks replaced HOC/render-props, single `@apollo/client` package replaced the scattered `apollo-*` packages). Later major versions have reorganized entry points and trimmed legacy APIs; pin versions and read the migration guide rather than trusting SemVer minors, since the project's [versioning policy](https://github.com/apollographql/apollo-client/blob/main/VERSIONING_POLICY.md) permits some breaking-adjacent changes (transpilation targets, dependency support) in minors.

## When to Use / When Not

**Use when:**
- You are building a React app over GraphQL with an interconnected object graph that benefits from normalized, cross-query cache updates.
- You want declarative data loading, optimistic UI, and Suspense integration without hand-rolling a cache.
- You use subscriptions, local state, and remote state through one consistent API.

**Avoid when:**
- Your needs are "fetch this query, render it" — a document cache (TanStack Query) or a minimal client is far lighter and simpler.
- Bundle size is a hard constraint.
- You do not want to reason about normalization, `keyFields`, and `merge` functions to get pagination and updates correct.
- Your backend is REST; GraphQL-shaped tooling adds friction with no cache payoff.

## Alternatives

- urql — lighter, modular GraphQL client with an exchange pipeline; document cache by default, normalized cache opt-in. Use when you want GraphQL without Apollo's weight or complexity.
- TanStack/query (react-query) + graphql-request — generic async-state + a tiny GraphQL fetcher; a document cache, not normalized. Use when you want one caching model across REST and GraphQL and don't need cross-query normalization.
- facebook/relay — compiler-driven, strict, highest-performance normalized client. Use when the team accepts Relay's conventions and schema requirements in exchange for its guarantees at scale.
- prisma-labs/graphql-request — minimal fetch wrapper, no cache. Use for scripts, server-to-server calls, or when you bring your own caching.
- vercel/swr — generic data fetching/revalidation. Use when you want a small, opinion-light cache and are not tied to GraphQL semantics.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-2017 | Successor to Redux-based `react-apollo`; original `apollo-client`. |
| 2.0 | 2017-10 | Pluggable `ApolloLink` architecture, `apollo-cache-inmemory`. |
| 3.0 | 2020-07 | Single `@apollo/client` package, hooks-first, reactive variables, rewritten InMemoryCache[^1]. |
| 3.8 | 2023 | `useSuspenseQuery`, `useFragment`, `useBackgroundQuery` (React 18 Suspense). |
| 3.x | 2023-2025 | Incremental React 18/19, Suspense, and cache improvements across minors. |

Development remains active, with the three-person core team (Jeff Auriemma, Jerel Miller, Lenz Weber-Tronic) shipping on `main` continuously[^3]. Consult the repository's [roadmap](https://github.com/apollographql/apollo-client/blob/main/ROADMAP.md) and releases for the current major line.

## References

[^1]: Apollo Client 3.0 release — normalized cache rewrite and unified `@apollo/client` package. https://www.apollographql.com/blog/announcement/frontend/apollo-client-3-0-now-available/
[^2]: Apollo Client Next.js / RSC integration and its constraints. https://www.apollographql.com/docs/react/integrations/nextjs
[^3]: Apollo Client repository maintainers and roadmap. https://github.com/apollographql/apollo-client

## Tags

graphql, graphql-client, typescript, javascript, react, caching, data-fetching, apollo, frontend, state-management, normalized-cache
