# trpc/trpc

> End-to-end typesafe APIs for TypeScript, with no schema, no code generation, and no runtime contract — the types are the contract.

[GitHub repo](https://github.com/trpc/trpc) ·
[Official website](https://trpc.io) ·
[License: MIT](https://github.com/trpc/trpc/blob/main/LICENSE)

## Overview

tRPC lets a TypeScript client call TypeScript server procedures with full static type
inference — input types, output types, and error shapes — without generating any
client code or declaring a separate schema (no `.proto`, no GraphQL SDL, no OpenAPI
document). The client imports only the server router's *type*, never its runtime code,
and TypeScript resolves the call signatures across the boundary[^1]. First released in
2020 by Alex Johansson (KATT), it became one of the anchor tools of the "TypeScript
full-stack" movement, particularly the T3 stack (Next.js + tRPC + Prisma).

The defining tradeoff follows directly from the mechanism: because the contract is a
shared TypeScript type, both ends must be TypeScript and must compile against the same
types — in practice the same repository or a shared workspace package. tRPC gives you
zero-overhead type safety exactly when client and server are one codebase, and gives you
nothing when they are not. It is not an interoperability protocol; it is a way to erase
the API layer inside a monorepo. This is why it competes with GraphQL and REST-codegen
tools on developer experience but cannot replace them for public, multi-language, or
independently versioned APIs.

The library is deliberately small: `@trpc/server` and `@trpc/client` have zero runtime
dependencies, and the wire protocol is plain HTTP (JSON by default). The React bindings,
however, are a wrapper over TanStack Query, so most real usage inherits that library's
model and its learning curve.

## Getting Started

```sh
npm install @trpc/server @trpc/client
# React bindings (optional): @trpc/react-query @tanstack/react-query
# input validation (any Standard Schema lib): zod
```

```ts
// server/trpc.ts — define a router
import { initTRPC } from "@trpc/server";
import { z } from "zod";

const t = initTRPC.create();

export const appRouter = t.router({
  greet: t.procedure
    .input(z.object({ name: z.string() }))
    .query(({ input }) => `Hello, ${input.name}`),
});

export type AppRouter = typeof appRouter; // export the TYPE only
```

```ts
// client.ts — consume it, fully typed, no codegen
import { createTRPCClient, httpBatchLink } from "@trpc/client";
import type { AppRouter } from "./server/trpc";

const trpc = createTRPCClient<AppRouter>({
  links: [httpBatchLink({ url: "http://localhost:3000/trpc" })],
});

const msg = await trpc.greet.query({ name: "Tom" }); // string, checked at compile time
```

## Architecture / How It Works

A tRPC API is a **router** of **procedures**. Each procedure is a `query` (read),
`mutation` (write), or `subscription` (stream). Procedures are composed from a base
`t.procedure` through a builder chain — `.input()`, `.output()`, `.use()` (middleware),
`.query()/.mutation()` — and routers nest into a single root router whose type is the
entire API surface.

- **Input validation** is delegated to a validator library. tRPC does not ship a schema
  system; it accepts any validator that conforms to Standard Schema (Zod, Valibot,
  ArkType, etc.), and the validator's *inferred* type becomes the procedure's input
  type[^2]. Standard Schema support landed in v11; earlier versions had per-library
  adapters.
- **Links** are the client-side middleware pipeline (conceptually like Apollo Links).
  `httpBatchLink` collapses multiple calls fired in the same tick into one HTTP request;
  `httpLink` sends one request each; `splitLink` routes by condition; `wsLink` /
  `httpSubscriptionLink` handle subscriptions. Batching is on by default in most starters
  and materially changes your request count and server routing.
- **Adapters** bind the router to a host: Next.js (App and Pages routers), Express,
  Fastify, AWS Lambda, the Fetch/`Request` standard (Cloudflare Workers, Deno, Bun,
  edge), or a standalone Node server.
- **Serialization** is JSON by default. A "transformer" (commonly SuperJSON) is needed to
  transport `Date`, `Map`, `Set`, `BigInt`, etc. The transformer must be configured
  identically on both ends or types silently mismatch at runtime.
- **React** integration (`@trpc/react-query`) generates a hook tree mirroring the router
  (`trpc.greet.useQuery(...)`) that delegates caching, refetching, and mutation state to
  TanStack Query. You are effectively using React Query with typed keys.

There is no intermediate artifact. That is the whole point and the whole constraint:
delete the shared type and there is nothing left describing the API.

## Production Notes

**TypeScript compiler performance is the real scaling limit.** Large routers (hundreds
of procedures, deeply nested, heavy generic inference) slow down `tsc` and, more
painfully, the in-editor language server — autocomplete lag and slow "go to definition"
are the common symptoms. Mitigations: split into sub-routers, avoid extremely deep
router nesting, prefer `interface` over large inline types, and keep validator schemas
from ballooning. This is inherent to inference-as-contract, not a bug that gets fixed.

**Client and server must share types at build time.** For a monorepo this is trivial; for
separately deployed services or a public API it is a wall. Third-party consumers, mobile
apps in other languages, or teams that version their API independently cannot use the
tRPC client. The escape hatch is `trpc-openapi` / `trpc-to-openapi` to expose REST +
OpenAPI alongside, but that reintroduces the schema layer tRPC was meant to remove.

**Batching has operational consequences.** With `httpBatchLink`, several logical calls
arrive as one HTTP POST, so per-endpoint rate limiting, caching (a batched request is not
cacheable per-procedure), logging, and CDN behavior all see one opaque request. Teams
that need per-procedure HTTP semantics switch specific calls to `httpLink` via
`splitLink`.

**Subscriptions.** v11 added SSE-based subscriptions (`httpSubscriptionLink`) which are
easier to deploy through standard HTTP infrastructure than WebSockets, but long-lived
streaming still needs a host that supports it — many serverless/edge platforms cap
connection duration.

**Transformer drift and error handling.** A mismatched or missing transformer produces
values that typecheck but deserialize wrong (a `Date` arriving as a string). Errors cross
the boundary as `TRPCError` with a code; the `errorFormatter` shapes what the client
sees, and it is easy to leak internal messages if left at defaults.

**Upgrade pain — v9 to v10 was a rewrite.** v10 (2022) replaced the string-path API with
the chained builder and changed nearly every import; it shipped a `@trpc/upgrade` /
interop layer, but migration was substantial. v10 to v11 (2025) was far smaller but bumped
the React bindings to require `@tanstack/react-query` v5[^3].

## When to Use / When Not

**Use when:**
- Client and server are both TypeScript in one repo or shared workspace.
- You want typed API calls without maintaining a GraphQL schema or OpenAPI spec.
- You're building a Next.js / T3-style full-stack app and control both ends.
- You value inference-driven refactors: rename a procedure, get compile errors at every call site.

**Avoid when:**
- The API is public or consumed by non-TypeScript clients (mobile native, other backends) — use a schema-based protocol.
- Client and server are versioned or deployed independently.
- You need a language-agnostic contract or generated SDKs — GraphQL, gRPC/Connect, or OpenAPI codegen fit better.
- Your router is huge and editor responsiveness already suffers from TS inference cost.

## Alternatives

- graphql/graphql-js (+ Apollo/urql) — schema-first, language-agnostic, introspectable; use when consumers are external or polyglot.
- connectrpc/connect-es — Protobuf contracts over HTTP, browser + backend + cross-language; use when you need a real IDL and multi-language clients.
- ts-rest — typesafe REST with an explicit shared contract object; use when you want tRPC-like DX but decoupled client/server and plain REST URLs.
- honojs/hono — its built-in RPC client gives typed calls from a Hono server; use when Hono is already your server framework.
- openapi-typescript / orval — generate typed clients from an OpenAPI document; use when the server is the source of truth and may not be TypeScript.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2020-07 | Initial release; string-path procedure API[^1]. |
| 9.x | 2021 | Widely adopted era of the pre-rewrite API; popularized by the T3 stack. |
| 10.0 | 2022 | Major rewrite: chained procedure builder, inferred router type, new imports[^4]. |
| 11.0 | 2025 | Standard Schema validators, SSE subscriptions, non-JSON/FormData & streaming; requires TanStack Query v5[^3]. |

## References

[^1]: tRPC documentation — concepts and quickstart. https://trpc.io/docs
[^2]: Standard Schema — shared validator interface (Zod, Valibot, ArkType). https://standardschema.dev
[^3]: tRPC blog, "Announcing tRPC v11". https://trpc.io/blog/announcing-trpc-v11
[^4]: tRPC v10 migration guide (from v9). https://trpc.io/docs/migrate-from-v9-to-v10

## Tags

typescript, rpc, api, type-safety, full-stack, react, nextjs, tanstack-query, monorepo, web-framework, backend
