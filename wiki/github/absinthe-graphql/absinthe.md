# absinthe-graphql/absinthe

> The GraphQL toolkit for Elixir — a schema-first, macro-defined, compile-time-verified GraphQL server built on the BEAM.

[GitHub repo](https://github.com/absinthe-graphql/absinthe) ·
[Official website](https://absinthe-graphql.org) ·
[License: MIT](https://github.com/absinthe-graphql/absinthe/blob/main/LICENSE.md)

## Overview

Absinthe is the de facto GraphQL implementation for Elixir, first released in 2015 by Bruce Williams and Ben Wilson[^1]. Its parser was originally derived from the earlier `graphql-elixir/graphql-elixir` project[^2], which Absinthe has long since outgrown and replaced as the ecosystem standard. As of 2026 it is effectively the only maintained GraphQL server for the language — there is no serious competing Elixir library, so "GraphQL on the BEAM" and "Absinthe" are nearly synonymous.

The defining choice is **schema-as-code via compile-time macros**. You declare types with `object`, `field`, `query`, and friends inside a module; the macros build and validate the schema structure at compile time, so malformed schemas fail the build rather than the first request. This buys runtime performance and early error detection at the cost of longer compile times and a schema that lives in Elixir modules rather than an SDL file (though `import_sdl` lets you define schemas from GraphQL SDL strings when you prefer that).

Absinthe leans hard on the BEAM's concurrency model. Field resolution can run asynchronously across processes for free, and subscriptions ride on Phoenix's PubSub layer. The tradeoff running through the whole library: it is a small core (`absinthe`) surrounded by separate companion packages you must assemble yourself — HTTP transport, Phoenix integration, batching, and Relay support each live in their own hex package.

## Getting Started

```elixir
# mix.exs — the core package plus HTTP transport
def deps do
  [
    {:absinthe, "~> 1.7"},
    {:absinthe_plug, "~> 1.5"}
  ]
end
```

Absinthe requires Elixir 1.10 or higher[^3].

```elixir
defmodule MyAppWeb.Schema do
  use Absinthe.Schema

  object :user do
    field :id, :id
    field :name, :string
    field :email, :string
  end

  query do
    field :user, :user do
      arg :id, non_null(:id)

      resolve fn %{id: id}, _ ->
        {:ok, MyApp.Accounts.get_user(id)}
      end
    end
  end
end
```

A resolver returns `{:ok, value}` or `{:error, reason}`. Note that schema field names use idiomatic Elixir `snake_case`; Absinthe transparently translates to `camelCase` for clients via its default adapter.

## Architecture / How It Works

Absinthe processes every document as a series of **phases operating on a Blueprint**. `Absinthe.Pipeline` is an ordered list of phase modules; the Blueprint is the internal IR that starts as parsed AST and is progressively decorated with type info, validation results, and resolution values. Because the pipeline is just a list, you can insert, remove, or swap phases — including per-document — which is how features like persisted queries and custom validations are built.

The stages, roughly:

1. **Parse** — the GraphQL document string becomes a Blueprint AST (a Leex/Yecc-generated lexer and parser).
2. **Validate** — a suite of phases check the document against the compiled schema (unknown fields, type mismatches, fragment rules, and so on), producing detailed errors.
3. **Resolve** — each field runs through a **middleware chain**. `resolve fn ... end` is sugar for appending a resolver function to that field's middleware list; the default for an undecorated field is a map/struct key lookup. Middleware can short-circuit, transform, or defer.
4. **Result** — the resolved Blueprint is serialized to the response map.

**Batching and N+1.** Naive resolvers issue one query per field per parent, which produces the classic N+1 problem. Absinthe ships `Absinthe.Middleware.Batch` for aggregating calls, but the idiomatic solution is the separate `dataloader` package (same org), which provides source-based batched loading integrated as middleware.

**Subscriptions.** `Absinthe.Subscription` implements GraphQL subscriptions over `Phoenix.PubSub`. Real-time delivery requires the `absinthe_phoenix` package and a Phoenix endpoint; documents are re-resolved and pushed when you call `Absinthe.Subscription.publish/3` with a matching topic.

**Transport is external.** The core library knows nothing about HTTP. `absinthe_plug` provides the Plug/Phoenix request handler and a built-in GraphiQL interface; `absinthe_phoenix` wires subscriptions into a Phoenix socket. This decomposition is deliberate ("small parts that do one thing well") but means a working API is always several packages.

## Production Notes

**Compile times scale with schema size.** Because the schema is macro-expanded and verified at compile time, large schemas (hundreds of types) noticeably lengthen builds and trigger recompilation when the schema module or its dependencies change. Splitting types across modules with `import_types` helps organization but not the fundamental cost.

**N+1 is opt-out, not opt-in.** A schema written with plain `resolve` functions that call the database will issue a query per field. You must reach for `dataloader` or batch middleware deliberately; nothing warns you at compile time that a resolver is about to fan out. This is the single most common performance surprise in production Absinthe.

**Complexity limiting is manual.** Absinthe supports query complexity analysis (`Absinthe.Complexity`) and a `max_complexity` option, plus depth constraints, but they are off by default. A public GraphQL endpoint without complexity/depth limits is exposed to expensive nested queries; configure these before exposing the API externally.

**Subscriptions need distributed PubSub to scale.** On a single node subscriptions work out of the box, but across a cluster you need a distributed `Phoenix.PubSub` adapter, and every publish re-runs resolution for matching subscribers — a high-fanout topic can become a resolution-cost multiplier. Budget for this rather than assuming subscriptions are free.

**Version cadence is uneven.** The v1.7 line held for roughly four years (v1.7.0 shipped January 2022) with only patch releases, so most existing production deployments are on 1.7.x. A burst of minor releases followed — v1.8.0 (Nov 2025), v1.9.0 (Nov 2025), v1.10.0 (Apr 2026), and v1.11.0 (Jun 2026)[^4] — so upgrade planning that assumed a slow-moving library needs revisiting; read the CHANGELOG for each step.

**Error shaping.** Resolvers return `{:error, ...}` where the error can be a string, keyword list, or map; Absinthe maps these into the GraphQL `errors` array. Getting consistent error contracts (codes, user-facing vs internal messages) across a large schema usually means a shared error-handling middleware, not per-resolver ad hoc returns.

## When to Use / When Not

**Use when:**
- You are building a GraphQL API on Elixir/Phoenix and want the ecosystem-standard, actively maintained library.
- You want compile-time schema verification and are willing to trade build time for it.
- You need real-time subscriptions and are already on Phoenix/PubSub.
- You value the pluggable pipeline (custom validations, persisted queries, per-document middleware).

**Avoid when:**
- A plain REST or JSON API meets your needs — GraphQL's schema, resolver, and N+1 overhead is not free.
- You are not on the BEAM; there is no reason to pick Absinthe outside Elixir/Erlang.
- You want a single batteries-included package — Absinthe is deliberately several packages you assemble.
- Your team is new to both GraphQL and Elixir; the compile-time macro model plus resolver batching is a steep combined learning curve.

## Alternatives

- rmosolgo/graphql-ruby — use instead when your backend is Ruby/Rails; comparable schema-class approach and maturity.
- 99designs/gqlgen — use instead on Go; schema-first with generated type-safe resolvers.
- graphql-python/graphene / strawberry — use instead on Python; code-first schema definition.
- graphql/graphql-js — use instead on Node.js; the reference JavaScript implementation.
- phoenixframework/phoenix — use instead when you want a plain REST/JSON API on Elixir without GraphQL's overhead.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-10 | First release; parser derived from graphql-elixir[^1][^2]. |
| 1.4.0 | 2017-11 | Pre-1.5 line; pipeline/blueprint architecture maturing. |
| 1.5.0 | 2020-05 | SDL-based schema definition (`import_sdl`) and other refinements. |
| 1.7.0 | 2022-01 | Long-lived stable line; most production deployments run 1.7.x[^4]. |
| 1.8.0 | 2025-11 | First minor after the ~4-year 1.7 era[^4]. |
| 1.9.0 | 2025-11 | Follow-on minor. |
| 1.10.0 | 2026-04 | Continued rapid cadence. |
| 1.11.0 | 2026-06 | Latest minor as of this writing[^4]. |

## References

[^1]: Absinthe project — GitHub repository and history. https://github.com/absinthe-graphql/absinthe
[^2]: LICENSE.md notes the parser is derived from GraphQL Elixir (Josh Price). https://github.com/absinthe-graphql/absinthe/blob/main/LICENSE.md
[^3]: Absinthe README — "Absinthe requires Elixir 1.10 or higher." https://github.com/absinthe-graphql/absinthe#installation
[^4]: Release tags — v1.7.0 (2022-01-19), v1.8.0 (2025-11-06), v1.9.0 (2025-11-21), v1.10.0 (2026-04-03), v1.11.0 (2026-06-04). https://github.com/absinthe-graphql/absinthe/tags

## Tags

elixir, graphql, api, beam, phoenix, schema-first, subscriptions, relay, backend, server, dataloader
