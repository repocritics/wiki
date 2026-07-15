# better-auth/better-auth

> Framework-agnostic, database-owning authentication for TypeScript — the library that treats email/password, sessions, and 2FA as core rather than plugins you bolt on.

[GitHub repo](https://github.com/better-auth/better-auth) ·
[Official website](https://better-auth.com) ·
[License: MIT](https://github.com/better-auth/better-auth/blob/main/LICENSE.md)

## Overview

Better Auth is a TypeScript authentication and authorization framework created by
Bereket Engida, first published in mid-2024[^1]. Its thesis is that auth in the
TypeScript ecosystem was a "half-solved problem": most libraries handled OAuth
sign-in but left email/password, session storage, account linking, 2FA, and
organization/multi-tenancy as exercises for the application developer or as a
reason to buy a hosted product. Better Auth ships those as first-class,
type-inferred features and owns the database schema they require.

The defining design choice — and the central tradeoff — is that Better Auth is
**database-first**. It is not a hosted service and not a stateless JWT helper; it
expects to create and manage `user`, `session`, `account`, and `verification`
tables (plus per-plugin tables) in your own database, reached through an adapter
(built-in Kysely SQL, or Prisma / Drizzle / MongoDB)[^2]. This gives you full
data ownership and server-side session revocation for free, at the cost of
running migrations and reconciling its schema with any user table you already
have. It is framework-agnostic by mounting a single standard-Request handler,
with first-party integrations for Next.js, Nuxt, SvelteKit, SolidStart, TanStack
Start, Astro, Remix, Hono, and Express.

The project grew unusually fast for an auth library — tens of thousands of stars
within roughly a year — which is both a signal of real demand and a caution: much
of the 0.x line saw frequent API changes, and the plugin surface is large enough
that individual plugins vary in maturity.

## Getting Started

```bash
npm install better-auth
```

```ts
// auth.ts — server instance
import { betterAuth } from "better-auth";
import { Pool } from "pg";

export const auth = betterAuth({
  database: new Pool({ connectionString: process.env.DATABASE_URL }),
  emailAndPassword: { enabled: true },
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
  },
});
```

```ts
// app/api/auth/[...all]/route.ts — Next.js App Router mount
import { auth } from "@/auth";
import { toNextJsHandler } from "better-auth/next-js";

export const { GET, POST } = toNextJsHandler(auth);
```

```ts
// client.ts — typed client, methods inferred from server config + plugins
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient();
await authClient.signIn.email({ email, password });
```

Generate the schema with the CLI: `npx @better-auth/cli generate` (emits schema
or migrations), then `npx @better-auth/cli migrate` for the Kysely adapter[^3].

## Architecture / How It Works

`betterAuth(config)` returns an object whose `.handler` accepts a standard `Request`
and returns a `Response`. Every framework adapter (`toNextJsHandler`, the Hono/Express
middleware, etc.) is a thin shim mounting that handler at `/api/auth/*`. All routes —
sign-in, callbacks, session, plugin endpoints — live under that single mount point.

Three layers matter:

- **Database adapters.** Core logic is written against an adapter interface. Kysely
  is built in and speaks SQLite/Postgres/MySQL directly; Prisma, Drizzle, and MongoDB
  are separate adapters. Only the Kysely path gets the CLI's `migrate` command — with
  Prisma/Drizzle you run `generate` to emit their schema and then use their own
  migration tooling[^3].
- **Sessions.** Sessions are rows in the `session` table keyed by a cookie. By
  default every authenticated request does a database lookup, which makes revocation
  instant but adds a query per request. An optional cookie cache signs a short-lived
  copy of the session into the cookie to skip that lookup, trading revocation latency
  for throughput.
- **Plugins.** Plugins (2FA, passkey/WebAuthn, magic link, `organization` for
  multi-tenancy, `admin`, `oidcProvider`, `sso`, Stripe, and more) can add endpoints,
  extend the schema with new tables/columns, and add methods to both server and
  client. Because the client is generated from the server's type, adding a server
  plugin surfaces its client methods with full type inference and no manual wiring.

The end-to-end type inference — server config flowing into client call signatures —
is the framework's most distinctive property and the reason its API feels cohesive
despite the breadth.

## Production Notes

**It owns a schema.** On a greenfield database this is painless. On an existing app
with a legacy `users` table, you must map Better Auth's expected columns, add its
`session`/`account`/`verification` tables, and decide whether to migrate old password
hashes. Budget real time for the integration; this is the most common friction point.

**Migration tooling is adapter-dependent.** The `migrate` CLI command only applies to
the Kysely adapter. Prisma and Drizzle users generate schema from Better Auth and then
own the migration lifecycle themselves — a step that surprises teams expecting one
command to do everything.

**Session strategy is a real decision.** Default DB-backed sessions cost one query per
request but give immediate server-side logout/revocation. Enabling the cookie cache
removes that query but means a revoked session stays valid until the cache TTL expires.
Pick deliberately based on whether instant revocation or per-request latency matters more.

**CSRF / origins.** Cross-origin and non-default deployment setups require configuring
`trustedOrigins` (and correct `baseURL`); getting this wrong produces sign-in requests
that are silently rejected as untrusted. Read the security config before deploying
behind a proxy or on a split frontend/backend domain.

**Version churn.** The library moved fast through 0.x and continues to iterate. Pin
exact versions, keep server and client packages in lockstep, and read release notes
before upgrading — plugin schemas occasionally change, which means new migrations.

**Secrets.** A stable `BETTER_AUTH_SECRET` is required in production; rotating it
invalidates existing signed cookies/sessions, so plan rotation as a logout event.

## When to Use / When Not

**Use when:**
- You want to own your auth data in your own database and avoid a per-MAU billing model.
- You need more than social login — email/password, 2FA, passkeys, organizations,
  admin, or an OIDC provider — without stitching separate libraries together.
- You value end-to-end TypeScript inference from server config to client calls.
- You're spanning multiple frameworks or a non-Next.js stack (Hono, SvelteKit, Nuxt).

**Avoid when:**
- You want a fully managed, no-database, no-migrations auth service (use a hosted
  provider instead).
- You cannot tolerate the schema-ownership integration cost against a large legacy
  user table.
- You need a battle-tested, slow-moving dependency with a multi-year stability track
  record — this is a young, fast-moving project.
- You need a language-neutral identity server that non-TypeScript services also consume.

## Alternatives

- nextauthjs/next-auth — the established Auth.js option; use it when you're Next-centric
  and prefer stateless/OAuth-first sessions without owning a full auth schema.
- lucia-auth/lucia — a minimal, unbundled approach (now positioned as a learning
  reference rather than a maintained dependency); use its patterns when you want to
  hand-roll sessions with full control.
- ory/kratos — a standalone, self-hosted identity server; use when non-TypeScript
  services must share one auth system over an API.
- keycloak/keycloak — heavyweight Java IAM with SSO/federation; use for enterprise
  realms, SAML, and centralized admin.
- supabase/supabase — auth as part of a managed Postgres backend; use when you want
  hosted auth bundled with database and storage.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial publish | 2024-05 | First public commit; framework-agnostic TS auth[^1]. |
| 0.x line | 2024 | Rapid iteration — adapters, plugin system, client inference; frequent breaking changes. |
| 1.0 (stable) | late 2024 | First stable API surface after the 0.x churn[^4]. |
| 1.x | 2025–2026 | Expanding plugin ecosystem (SSO, OIDC provider, Stripe, organizations). |

## References

[^1]: Better Auth repository, created 2024-05-19. https://github.com/better-auth/better-auth
[^2]: Better Auth docs, "Database" and adapters (Kysely / Prisma / Drizzle / MongoDB). https://better-auth.com/docs/concepts/database
[^3]: Better Auth docs, "CLI" — `generate` and `migrate`. https://better-auth.com/docs/concepts/cli
[^4]: Better Auth documentation and releases. https://github.com/better-auth/better-auth/releases

## Tags

typescript, authentication, authorization, oauth2, oidc, sso, session-management, iam, framework-agnostic, plugin-architecture
