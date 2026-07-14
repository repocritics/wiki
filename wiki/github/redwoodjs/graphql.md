# redwoodjs/graphql

> The opinionated, full-stack React + GraphQL + Prisma framework formerly known as RedwoodJS — now in maintenance-forward mode as its team pivots to RedwoodSDK.

[GitHub repo](https://github.com/redwoodjs/graphql) ·
[Official website](https://redwoodjs.com) ·
[License: MIT](https://github.com/redwoodjs/graphql/blob/main/LICENSE)

## Overview

Redwood is a batteries-included, TypeScript-first web framework created by Tom
Preston-Werner (GitHub co-founder) and collaborators, first announced publicly in
early 2020 and reaching 1.0 in March 2022[^1]. Its thesis: pick the best pieces of
the JS/TS ecosystem — React, Prisma, GraphQL, Jest, Storybook — and wire them into a
single opinionated monorepo so a solo developer or small team gets a full-stack app
with authentication, data fetching, testing, and deploy config already integrated.
It was one of the flagship "Jamstack" frameworks and is closely associated with
Netlify (also co-founded by Preston-Werner's circle).

The repository was originally `redwoodjs/redwood`; GitHub now redirects it to
`redwoodjs/graphql`. The rename reflects a strategic split: the classic GraphQL-based
framework documented here (internally the "Arapahoe" stable epoch, with a
next-generation "Bighorn" epoch in canary) is being wound down in favor of
**RedwoodSDK** — a separate, ground-up React-on-Cloudflare framework that starts as a
Vite plugin and targets Workers/D1/Durable Objects[^2]. Anyone evaluating "Redwood"
in 2026 must first decide which product they mean; the two share a brand and a Discord
but almost no code.

The defining tradeoff is opinionation. Redwood makes hundreds of decisions for you,
which is liberating on a greenfield app and constraining the moment you want to
deviate — swapping the GraphQL layer, the router, or the build tooling means fighting
the framework rather than configuring it.

## Getting Started

```bash
yarn create redwood-app my-app   # scaffolds web/ + api/ monorepo
cd my-app
yarn redwood dev                 # web on :8910, api (GraphQL) on :8911
```

```bash
# generate a Prisma-backed CRUD UI + SDL + service in one command
yarn redwood generate scaffold post
yarn redwood prisma migrate dev
```

```jsx
// web/src/components/BlogPostsCell/BlogPostsCell.jsx
// A "Cell" — Redwood's declarative data-fetching primitive.
export const QUERY = gql`
  query BlogPostsQuery {
    posts { id title }
  }
`

export const Loading = () => <div>Loading…</div>
export const Empty = () => <div>No posts yet</div>
export const Failure = ({ error }) => <div>Error: {error.message}</div>
export const Success = ({ posts }) =>
  posts.map((p) => <article key={p.id}>{p.title}</article>)
```

## Architecture / How It Works

A Redwood app is a Yarn-workspaces monorepo with two sides: `web/` (React frontend)
and `api/` (serverless GraphQL backend). The CLI (`redwood` / `rw`) ties them together
with generators, and a shared `redwood.toml` configures both.

- **Cells** are the signature abstraction. A Cell is a module that exports named
  `QUERY`, `Loading`, `Empty`, `Failure`, and `Success` components; Redwood compiles
  these into an Apollo Client query with all four UI states wired automatically. This
  eliminates the most common React data-fetching boilerplate but is Redwood-specific
  and does not transfer to other codebases.
- **Router.** Redwood ships its own `<Router>`/`<Route>` (not React Router), with
  route definitions in a single `Routes.jsx`, named routes, and set-level layouts.
- **GraphQL API.** The backend is SDL-first: you write `.sdl.ts` schema files and
  matching **services** (plain functions that act as resolvers). Redwood originally
  used Apollo Server, then migrated the server to **GraphQL Yoga + Envelop** for
  plugin-based middleware[^3]. Prisma sits underneath the services as the data layer.
- **Build tooling** moved from Webpack to **Vite** (default from v6, 2023), which cut
  cold dev-server times substantially. The Bighorn epoch pushes further toward React
  Server Components and Server Actions layered on Vite.
- **Deploy targets.** Because the API compiles to serverless functions, `rw setup
  deploy` supports Netlify, Vercel, Render, AWS (via Serverless/Fastify server mode),
  and others with minimal code change.

The strength is cohesion: generators, types, tests, and Storybook stories are produced
together and stay consistent. The coupling cost is that Cells, the router, and the
SDL/services convention are load-bearing — you rarely touch React data fetching or
Express directly, so the abstractions must hold for your use case.

## Production Notes

- **Project status is the biggest caveat.** The GraphQL framework is stable but its
  center of gravity has shifted to RedwoodSDK. Bighorn (the RSC-based next epoch) has
  no production release and ships only as canaries. Treat new-feature velocity here as
  low and plan around long-term maintenance, not rapid evolution.
- **Cold starts.** Splitting the API into serverless functions means the GraphQL
  endpoint pays serverless cold-start latency on low-traffic deploys. The "server"
  deploy mode (a long-running Fastify process) avoids this but gives up the pure
  serverless model Redwood was designed around.
- **Prisma coupling.** Redwood assumes Prisma; using a different ORM means abandoning
  scaffolds, the auth generators, and most of the tutorial-blessed path. Prisma's own
  migration and connection-pooling caveats (e.g. serverless connection exhaustion,
  needing a pooler like PgBouncer/Prisma Accelerate) become yours.
- **Upgrade friction across majors.** Redwood shipped roughly yearly majors (v1→v8)
  with codemods, but the Webpack→Vite transition (v6) and the GraphQL server swap
  required config and occasionally service changes. The Arapahoe→Bighorn direction is
  a larger break still.
- **Auth.** Redwood has a first-class auth story via `rw setup auth <provider>`
  (Auth0, Clerk, Supabase, Firebase, dbAuth self-hosted, and more) — one of its
  genuine strengths — but each provider's tokens and middleware are yours to operate.
- **Bundle/type-check cost.** Large SDL surfaces and generated types make TypeScript
  checking, not bundling, the dominant CI cost on big apps.

## When to Use / When Not

**Use when:**
- You want a full-stack React + GraphQL + Prisma app with auth, testing, and deploy
  config already integrated, and you like strong conventions.
- You're building a prototype or early-stage startup and value velocity over
  flexibility.
- You're comfortable adopting a framework whose future is explicitly the *separate*
  RedwoodSDK, and you want a stable GraphQL foundation in the meantime.

**Avoid when:**
- You need a framework under active feature development — the momentum is now in
  RedwoodSDK, and you'd likely start new projects there.
- You don't want GraphQL: Redwood's whole data layer assumes it (tRPC/REST stacks fit
  awkwardly).
- You need to swap core pieces (router, ORM, GraphQL server) — the opinionation fights
  you.
- You're targeting Cloudflare Workers as a first-class runtime: that is RedwoodSDK's
  design center, not this framework's.

## Alternatives

- redwoodjs/sdk — the same team's successor: React on Cloudflare via a Vite plugin,
  RSC and Server Functions. Use it when starting a new Redwood-flavored project today.
- remix-run/react-router — full-stack React (Remix folded into React Router 7); use
  when you want web-standard loaders/actions without a mandatory GraphQL layer.
- t3-oss/create-t3-app — Next.js + tRPC + Prisma + Tailwind; use when you want
  end-to-end type safety without GraphQL.
- blitz-js/blitz — "zero-API" full-stack React with a Rails-like ethos; use when you
  want Redwood's batteries but RPC instead of GraphQL.
- wasp-lang/wasp — full-stack React/Node from a config DSL; use when you want even more
  generation and less hand-wiring.

## History

| Version | Date | Notes |
|---------|------|-------|
| announce | 2020-02 | RedwoodJS revealed publicly by Tom Preston-Werner[^1]. |
| 1.0 | 2022-03 | First stable release after a long 0.x preview[^1]. |
| 3.0 | 2022 | GraphQL server migrated toward Yoga/Envelop[^3]. |
| 6.0 | 2023 | Vite replaces Webpack as the default bundler. |
| 7.0 | 2024 | Continued TS-first hardening, trusted documents. |
| 8.0 | 2024 | Latest stable "Arapahoe" major line. |
| — (Bighorn) | 2025–26 | Canary-only RSC/Server-Actions epoch; no GA. |
| rename | 2026 | Repo `redwood` → `graphql`; focus shifts to RedwoodSDK[^2]. |

## References

[^1]: RedwoodJS blog and release history — "Redwood 1.0" (2022-03). https://redwoodjs.com/blog
[^2]: RedwoodSDK — React on Cloudflare. https://github.com/redwoodjs/sdk · https://rwsdk.com
[^3]: GraphQL Yoga / Envelop, Redwood's GraphQL server stack. https://the-guild.dev/graphql/yoga-server · https://the-guild.dev/graphql/envelop
[^4]: Redwood documentation — Cells, Router, Services, Deploy. https://redwoodjs.com/docs/introduction

## Tags

typescript, react, graphql, prisma, full-stack, framework, jamstack, apollo, serverless, monorepo
