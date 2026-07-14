# graphql/graphql-js

> The reference implementation of GraphQL for JavaScript — the parser, type system, validator, and executor that most JS GraphQL servers are built on top of.

[GitHub repo](https://github.com/graphql/graphql-js) ·
[Official website](https://graphql.org/graphql-js/) ·
[License: MIT](https://github.com/graphql/graphql-js/blob/main/LICENSE)

## Overview

GraphQL.js is the canonical JavaScript implementation of the GraphQL specification, released by Facebook in July 2015 alongside the spec itself[^1]. It is not a server, a client, or a framework — it is the library that turns a GraphQL schema and a query string into a result. Almost every JS GraphQL server (Apollo Server, GraphQL Yoga, Mercurius, the now-deprecated express-graphql) delegates parsing, validation, and execution to this package, which ships on npm simply as `graphql`.

The project is stewarded by the GraphQL Foundation under the Linux Foundation, and contributions require signing a Specification Membership agreement enforced by an EasyCLA bot[^2]. Because it is the reference implementation, its behavior is effectively normative: when the spec is ambiguous, what graphql-js does tends to become what "correct" means in practice. This gives it unusual stability — the public surface has changed slowly across a decade — but also makes it conservative about adding conveniences that belong in higher layers.

The defining tension is scope. graphql-js deliberately stops at the schema/execution boundary and leaves HTTP transport, batching, caching, subscriptions, and auth to the ecosystem. That keeps the core small and unopinionated, but means a working server is always graphql-js *plus* several other packages, and newcomers routinely mistake the library for a batteries-included server framework.

## Getting Started

```sh
npm install graphql
```

```js
import {
  graphql,
  GraphQLSchema,
  GraphQLObjectType,
  GraphQLString,
} from 'graphql';

const schema = new GraphQLSchema({
  query: new GraphQLObjectType({
    name: 'Query',
    fields: {
      hello: {
        type: GraphQLString,
        resolve: () => 'world',
      },
    },
  }),
});

const result = await graphql({ schema, source: '{ hello }' });
console.log(result); // { data: { hello: 'world' } }
```

`graphql()` parses the source, validates it against the schema, then executes — reporting errors instead of throwing when the query is invalid. Schemas can also be built from SDL with `buildSchema()`, though the programmatic constructor form above is what most tooling generates under the hood.

## Architecture / How It Works

The library is a pipeline of independent stages, each usable on its own:

1. **Lexer + parser** (`parse`) — turns a query string into a GraphQL AST. The AST node shapes are part of the public API and are consumed directly by codegen tools, linters, and editors.
2. **Type system** (`GraphQLSchema`, `GraphQLObjectType`, scalars, interfaces, unions, enums, input types) — an in-memory graph of type definitions with attached `resolve` functions.
3. **Validation** (`validate`) — runs a set of rules (fragment usage, field existence, argument types, etc.) against a query + schema before execution. Rules are pluggable.
4. **Execution** (`execute`) — walks the query against the schema, calling resolvers, coercing outputs, collecting errors, and assembling the response. Resolvers may return values, promises, or arrays of promises; execution awaits and flattens them.
5. **Introspection** — the schema can describe itself via the `__schema` meta-fields, which is how GraphiQL and client codegen discover types.

Every field resolves independently, which is the source of both GraphQL's flexibility and its most common performance failure: a naive resolver graph issues one backend call per field per node. There is no built-in batching; the standard fix is `DataLoader`[^3] (also from Facebook), wired into per-request context.

The library was originally written in Flow and migrated to TypeScript in v16 (2021)[^4], which changed how types are imported by downstream code. It publishes dual CommonJS (`.js`) and ESM (`.mjs`) builds selected through the `package.json` `exports` map, so tree-shaking bundlers pull in only the parts of the library a project uses.

## Production Notes

**No transport, no server.** graphql-js gives you `execute`; you supply the HTTP layer. The maintainers' recommended transport is now `graphql-http`[^5]; `express-graphql` is deprecated and unmaintained. Full servers (Apollo Server, GraphQL Yoga, Mercurius) bundle transport, subscriptions, and plugins on top.

**Denial-of-service is your problem.** The core does not limit query depth, breadth, or complexity. A single deeply nested or aliased query can fan out into an enormous execution tree. Production deployments generally add `graphql-depth-limit`, `graphql-query-complexity`, or persisted/allow-listed queries. Leaving introspection enabled in production is a debated exposure — convenient for tooling, but it hands attackers a full schema map.

**The N+1 trap.** Because each field resolves in isolation, a `users { posts { author { name } } }` query without batching multiplies database round-trips. DataLoader is effectively mandatory at scale, and forgetting to scope it per-request causes cross-request cache leaks.

**Partial errors.** Execution does not fail atomically. A resolver that throws produces a `null` in that position, an entry in the top-level `errors` array, and otherwise-successful `data`. Clients must be written to handle partial responses; treating `errors` as fatal discards valid data.

**Upgrade friction.** Major versions move slowly but carry real breaks: the Flow→TypeScript switch changed type imports, and v17 (in development on the `17.x.x` branch) reworks incremental delivery (`@defer`/`@stream`) and removes long-deprecated APIs. Pin the major version and read the migration notes — the version-support policy backports security fixes only to the current and previous major[^6]. There is no LTS line.

## When to Use / When Not

**Use when:**
- You are building a GraphQL server in Node and want the spec-correct core, or you are writing tooling (codegen, linters, gateways) that manipulates GraphQL ASTs and schemas directly.
- You need a stable, minimal dependency you can wrap in your own transport and conventions.
- You want behavior that tracks the specification exactly.

**Avoid when:**
- You want a ready-to-run server with auth, subscriptions, and caching — reach for GraphQL Yoga or Apollo Server, which use graphql-js internally.
- Your API is simple CRUD for a single first-party client — REST or an RPC layer like tRPC has far less ceremony.
- You cannot budget for query-cost limiting and per-request batching; without them a public GraphQL endpoint is a DoS liability.

## Alternatives

- apollographql/apollo-server — full-featured server (caching, plugins, federation) built on graphql-js; use when you want batteries included over assembling your own.
- dotansimha/graphql-yoga — lighter spec-compliant server on graphql-js and `graphql-http`; use when you want a modern default with less config than Apollo.
- ardatan/graphql-tools — SDL-first schema construction and schema stitching; use when you prefer defining schemas as text over the programmatic constructors.
- trpc/trpc — end-to-end typesafe RPC for TypeScript monorepos; use when client and server are both yours and you don't need GraphQL's cross-client query flexibility.
- graphql/graphql-http — the reference HTTP transport; use alongside graphql-js when you want a compliant server layer and nothing more.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.4.x | 2015-07 | First public release alongside the GraphQL spec[^1]. |
| 14.0 | 2018-08 | Modernized codebase; Flow types, error-handling changes. |
| 15.0 | 2020-04 | Stricter validation, execution refinements. |
| 16.0 | 2021-10 | Migrated from Flow to TypeScript; dual CJS/ESM output[^4]. |
| 17.x | in dev | `@defer`/`@stream` incremental delivery, deprecated-API removal (`17.x.x` branch)[^6]. |

## References

[^1]: Lee Byron, "GraphQL: A data query language" — Facebook, 2015-09-14 (public release July 2015). https://engineering.fb.com/2015/09/14/core-infra/graphql-a-data-query-language/
[^2]: graphql-js README, "Contributing" — Specification Membership agreement via EasyCLA. https://github.com/graphql/graphql-js#contributing
[^3]: DataLoader — batching/caching layer for GraphQL resolvers. https://github.com/graphql/dataloader
[^4]: graphql-js v16.0.0 release notes — TypeScript migration. https://github.com/graphql/graphql-js/releases/tag/v16.0.0
[^5]: graphql-http — reference-compliant GraphQL over HTTP. https://github.com/graphql/graphql-http
[^6]: graphql-js README, "Version Support". https://github.com/graphql/graphql-js#version-support

## Tags

graphql, javascript, typescript, api, query-language, schema, reference-implementation, server, nodejs, parser, execution-engine
