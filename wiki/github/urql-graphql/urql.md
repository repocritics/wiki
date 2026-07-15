# urql-graphql/urql

> A GraphQL client built around a composable middleware pipeline ("exchanges"), with document caching by default and normalized caching as an opt-in.

[GitHub repo](https://github.com/urql-graphql/urql) ·
[Official website](https://urql.dev) ·
[License: MIT](https://github.com/urql-graphql/urql/blob/main/LICENSE)

## Overview

urql is a GraphQL client originally built by Formidable and now maintained by
the urql GraphQL team, with Phil Pluckthun as the long-standing lead[^1]. It
positions itself as the lighter, more composable alternative to Apollo Client:
one small core (`@urql/core`) plus thin bindings for React, Preact, Vue, Svelte,
and Solid. As of 2026 it sits around 9k stars and is actively maintained
(last push July 2026), a smaller but stable presence next to Apollo's much
larger install base.

The defining design choice is the **exchange** — a middleware unit that sees
every operation flowing through the client and every result flowing back. Fetch,
caching, deduplication, auth, retries, and subscriptions are all exchanges you
compose into an ordered pipeline. This is urql's biggest strength (you add only
what you need, and can write your own behaviour) and its main source of
footguns (order matters, and the wrong order fails quietly).

The second defining choice is caching philosophy. Out of the box urql does
**document caching** — a coarse cache keyed by the `__typename`s a query touches,
invalidated wholesale when a mutation returns those types. This is simple and
requires no schema, but it is blunt. Normalized caching (the Apollo-style
entity graph) is available only by adding `@urql/exchange-graphcache`, which
brings back the configuration and manual cache-update burden urql's defaults
were designed to avoid.

## Getting Started

```bash
npm install urql graphql
```

```tsx
// React bindings. @urql/core is framework-agnostic; urql wraps it for React.
import { createClient, cacheExchange, fetchExchange, Provider, useQuery } from "urql";

const client = createClient({
  url: "https://api.example.com/graphql",
  exchanges: [cacheExchange, fetchExchange], // order matters, left-to-right
});

const TODOS = `query { todos { id title } }`;

function Todos() {
  const [result] = useQuery({ query: TODOS });
  const { data, fetching, error } = result;
  if (fetching) return <p>Loading…</p>;
  if (error) return <p>{error.message}</p>;
  return <ul>{data.todos.map((t) => <li key={t.id}>{t.title}</li>)}</ul>;
}

// <Provider value={client}><Todos /></Provider>
```

## Architecture / How It Works

The core is `@urql/core`, which is framework-agnostic and built on **Wonka**, a
lightweight streaming (push/pull) library also from Formidable[^2]. Every
operation is a stream event; every exchange is a stream transform. Understanding
Wonka is effectively a prerequisite for debugging non-trivial async behaviour or
authoring your own exchange.

An **exchange** is `(input) => output`, a function that receives the stream of
operations and returns the stream of results. The client threads operations
through the array you pass to `exchanges`, then back through them in reverse.
The canonical order is: dedup/synchronous exchanges → caching exchanges →
async "network" exchanges like `fetchExchange` last, because the fetch exchange
terminates the pipeline and does not forward. Common exchanges:

- `fetchExchange` — the terminating HTTP transport.
- `cacheExchange` — the default document cache.
- `@urql/exchange-graphcache` — normalized, schema-aware caching.
- `authExchange` — token attachment plus refresh-on-error; must sit before
  `fetchExchange`.
- `retryExchange`, `requestPolicyExchange`, `errorExchange`, `ssrExchange`,
  and a subscription exchange (typically over `graphql-ws`).

Framework bindings (`urql`, `@urql/preact`, `@urql/vue`, `@urql/svelte`,
`@urql/solid`) are thin: they wrap the same core client in each framework's
reactivity model (hooks, composables, stores, signals). `@urql/next` adds
Next.js App Router / RSC integration. Request policies (`cache-first`,
`cache-and-network`, `network-only`, `cache-only`) control cache vs. network on
a per-query basis and are the main knob most users reach for before touching
exchanges.

## Production Notes

**Document caching over-invalidates and under-invalidates.** Because the default
cache keys on `__typename`, a mutation returning a type wipes all cached queries
touching that type — even unrelated ones — while creations and deletions that
change *list membership* are invisible to it (a new item's type already existed).
Teams routinely hit "the list didn't update after adding an item" and reach for
`graphcache` or manual refetches.

**Graphcache is not free.** Normalized caching requires providing `keys`,
`resolvers`, and `updates`/`optimistic` config, and mutations that add or remove
list entries need explicit cache updates — the same class of work as Apollo's
`cache.modify`. It also adds meaningful bundle weight, eroding urql's size
advantage. Adopt it deliberately, not by default.

**Exchange ordering is a silent failure mode.** Placing `authExchange` after
`fetchExchange`, or a caching exchange after the transport, produces requests
with no auth or no caching and no error. The ordering rule (synchronous → cache
→ async → fetch last) has to be learned.

**v4 removed `dedupExchange`.** Deduplication was folded into the core, so v3
setups that imported and listed `dedupExchange` break on upgrade; the v4
migration is tracked in issue #3114[^3]. Client construction also standardized
on an explicit `exchanges` array.

**SSR and RSC are evolving surfaces.** Classic SSR uses `ssrExchange` to
serialize/rehydrate results. Next.js App Router support via `@urql/next` is newer
and has shifted with React Server Components; pin versions and test hydration.

**Subscriptions need wiring.** There is no built-in transport — add a
subscription exchange plus a WebSocket client (commonly `graphql-ws`).

**Docs domain moved.** Formidable's open-source work was reorganized under
Nearform, and documentation migrated from `formidable.com/open-source/urql` to
`urql.dev`; older links and blog posts may 404 or redirect.

## When to Use / When Not

**Use when:**
- You want a small GraphQL client and only pay for features you add.
- Your app is fine with document caching, or you'll adopt normalized caching
  deliberately and understand the cost.
- You need one client shared across React, Vue, Svelte, Solid, or Preact.
- You want to write custom transport/caching/auth logic as first-class middleware.

**Avoid when:**
- You want a normalized cache to "just work" out of the box with a large
  ecosystem of tooling — Apollo Client is the more batteries-included choice.
- You want a compiler-enforced, convention-heavy data layer for a very large
  app — Relay's strictness fits better.
- Your caching needs aren't GraphQL-specific — a transport-agnostic cache like
  TanStack Query over a plain fetcher may be simpler.

## Alternatives

- apollographql/apollo-client — use when you want a normalized cache by default,
  broad tooling, and are willing to accept a larger bundle and more config.
- facebook/relay — use when a large app benefits from a compiler, strict
  conventions (fragments, connections), and guaranteed data-masking.
- TanStack/query — use when your caching is not GraphQL-specific and you'd rather
  pair a generic async-state cache with a lightweight GraphQL fetcher.
- logaretm/villus — use when you're Vue-only and want an urql-inspired,
  Vue-native client with a similar plugin/pipeline model.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2018-01 | Repository initialized under Formidable. |
| 1.0 | 2019 | First stable release; React bindings and the exchange architecture[^1]. |
| 1.x | 2020 | Additional framework bindings and `@urql/exchange-graphcache` normalized caching. |
| 2.x | 2021 | Core/streaming refinements on Wonka. |
| 3.x | 2022 | API and exchange updates. |
| 4.0 | 2023 | `dedupExchange` removed (folded into core), explicit `exchanges`, TSDoc-first APIs; migration in #3114[^3]. |

## References

[^1]: urql README and project history — "founded by Formidable and is actively developed by the urql GraphQL team." https://github.com/urql-graphql/urql
[^2]: Wonka — the streaming library urql is built on. https://github.com/0no-co/wonka
[^3]: urql v4 migration guide (GitHub issue). https://github.com/urql-graphql/urql/issues/3114

## Tags

typescript, graphql, graphql-client, data-fetching, caching, react, vue, svelte, middleware, frontend
