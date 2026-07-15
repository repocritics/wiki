# nextauthjs/next-auth

> Authentication for the web — the most-deployed community auth library for Next.js, now the Auth.js project and, as of 2026, folded into Better Auth.

[GitHub repo](https://github.com/nextauthjs/next-auth) ·
[Official website](https://authjs.dev) ·
[License: ISC](https://github.com/nextauthjs/next-auth/blob/main/LICENSE)

## Overview

`next-auth` began in 2018 as a session/OAuth library scoped to Next.js[^1]. Over time the project generalized: the OAuth/OIDC and session machinery was extracted into a framework-agnostic core (`@auth/core`) built on standard Web `Request`/`Response`, and the umbrella was rebranded **Auth.js**, with sibling packages for SvelteKit, Express, SolidStart, and others[^2]. The `next-auth` package remains the Next.js binding and is by far the most-installed of the family — for most of the last five years it has been the default answer to "how do I add login to a Next.js app."

The defining trait is that it is a *library*, not a service: you run it inside your own app, own your user data, and it can operate entirely without a database (encrypted-JWT sessions) or with one via a pluggable adapter. That flexibility is also its main source of friction — the session-strategy split, the provider/callback model, and the v4-vs-v5 divide produce a large surface of "works on my machine but not in the edge runtime" issues.

The most important fact for anyone choosing this library in 2026: the project's own README now states that Auth.js has **joined Better Auth**, and steers new projects to Better Auth "unless there are some very specific feature gaps (most notably stateless session management without a database)"[^3]. Treat `next-auth` as a mature, maintained, but no-longer-strategically-advanced option.

## Getting Started

```bash
npm install next-auth
# App Router (v5 / Auth.js). Set AUTH_SECRET in your environment.
```

```ts
// auth.ts — v5 style: NextAuth() returns handlers + a universal auth() helper
import NextAuth from "next-auth";
import GitHub from "next-auth/providers/github";

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [GitHub], // reads AUTH_GITHUB_ID / AUTH_GITHUB_SECRET
});
```

```ts
// app/api/auth/[...nextauth]/route.ts
export { GET, POST } from "@/auth";

// Anywhere on the server — RSC, route handler, or server action:
import { auth } from "@/auth";
const session = await auth();
```

## Architecture / How It Works

The core is a set of route handlers mounted under one catch-all path (`/api/auth/*`). Requests flow through:

1. **Providers** — OAuth 2.0 / OIDC services, an email "magic link" provider, WebAuthn/passkeys, and a `Credentials` provider for hand-rolled username/password. Built-in configs exist for many popular services; each is a small descriptor of endpoints and profile mapping.
2. **Session strategy** — the pivotal design choice. `"jwt"` (the default) stores an encrypted JWE cookie (`A256CBC-HS512`) and keeps the server stateless. `"database"` stores an opaque session token and requires an **adapter**.
3. **Adapters** — a standardized CRUD interface (`getUser`, `createSession`, `linkAccount`, …) implemented for Postgres/MySQL/SQLite (Drizzle, Prisma, Kysely), MongoDB, DynamoDB, and others under `@auth/*-adapter` packages[^4].
4. **Callbacks** — `signIn`, `jwt`, `session`, `redirect` hooks let you gate logins and shape token/session contents. The `jwt` and `session` callbacks are where most real applications spend their configuration budget.

v5 (published as `next-auth@5` / Auth.js) is a rewrite on top of `@auth/core`. Its headline change is the single universal `auth()` function that replaces the v4 grab-bag (`getServerSession`, `getSession`, `useSession`, `withAuth` middleware) across RSCs, route handlers, server actions, and middleware[^2]. Config moves into a top-level `auth.ts`. The v4 client-side `<SessionProvider>` / `useSession` React hooks still exist for client components.

## Production Notes

The differentiators here are the footguns, and there are several well-known ones:

- **`Credentials` provider forces JWT sessions.** It cannot use the database session strategy — a persistent source of confusion when people expect DB-backed sessions after adding a password login.
- **Edge runtime and adapters conflict.** Most database adapters use Node drivers that do not run in Next.js middleware's edge runtime. The standard workaround is the **split-config pattern**: an `auth.config.ts` with only providers/callbacks (no adapter) imported by middleware, and a full `auth.ts` with the adapter used everywhere else[^5]. This is documented but non-obvious and trips up nearly every DB-backed v5 setup.
- **`AUTH_SECRET` is mandatory** (v4 called it `NEXTAUTH_SECRET`). Missing or rotated secrets silently invalidate every existing session cookie.
- **Long-lived beta.** `next-auth@5` sat in beta for a very long time; a large number of production apps ship on `5.0.0-beta.x` because there was no stable v5 to move to. v4 remains the "boring stable" choice.
- **Documentation churn.** The v4 → v5 rename, the NextAuth.js → Auth.js rebrand, and the `@auth/*` package split mean that a lot of blog posts, Stack Overflow answers, and even sub-pages of the official docs describe incompatible APIs. Verify which major version any snippet targets.
- **`redirect` / callback URL allow-listing.** Open-redirect mistakes are easy if you loosen the default same-origin `redirect` callback.

Session refresh is handled client-side by tab/window syncing and polling; short-session designs should tune `session.maxAge` and the `jwt` callback rather than assume server-side revocation (stateless JWT sessions cannot be force-revoked without a database or a denylist).

## When to Use / When Not

**Use when:**
- You have an existing Next.js app and want database-free, stateless sessions with minimal setup.
- You need broad built-in OAuth/OIDC provider coverage without wiring each service by hand.
- You want to own your user data and run auth inside your own deployment (no third-party identity vendor).
- You are already on `next-auth@4` in production and it works — there is no urgency to migrate.

**Avoid when:**
- You're starting a new project in 2026 — the maintainers now point new work at Better Auth[^3].
- You need server-side session revocation, richer session management, or a plugin ecosystem (2FA, org/teams, rate limiting) out of the box.
- You want stable, non-churning APIs and docs — the version/branding history makes onboarding noisy.
- You need auth across a non-JS backend or a hosted identity product with a UI and admin.

## Alternatives

- better-auth/better-auth — the project's own recommended successor; framework-agnostic, first-class database sessions, plugin system. Prefer for new projects unless you specifically want DB-free JWT sessions.
- lucia-auth/lucia — session-primitive approach where you own the auth code; use when you want full control (note the maintainer has been winding it down toward a learning resource rather than a dependency).
- supabase/auth — GoTrue-based hosted-style server; use when you're already on Supabase/Postgres and want managed auth.
- ory/kratos — full self-hosted identity server; use when you need enterprise identity flows, admin APIs, and language-agnostic backends.
- clerk / workos — hosted (non-OSS) identity products; use when you'd rather not run and secure auth yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2018 | First releases as NextAuth.js — Next.js-only session/OAuth library[^1]. |
| 4.0 | 2021 | The long-stable major; `getServerSession`, adapters ecosystem, encrypted JWT sessions[^4]. |
| Auth.js rebrand | 2022–2023 | `@auth/core` extracted; framework-agnostic packages (SvelteKit, Express, SolidStart); docs move to authjs.dev[^2]. |
| 5.0 (beta) | 2023– | Rewrite on `@auth/core`; universal `auth()` helper; config in `auth.ts`; extended beta period[^2]. |
| Better Auth | 2026 | Auth.js joins Better Auth; new projects steered to Better Auth[^3]. |

## References

[^1]: NextAuth.js / next-auth repository, created 2018-01-27. https://github.com/nextauthjs/next-auth
[^2]: Auth.js documentation (formerly NextAuth.js), `@auth/core` and framework packages. https://authjs.dev/
[^3]: nextauthjs/next-auth README, "Auth.js is now part of Better Auth" notice. https://better-auth.com/blog/authjs-joins-better-auth
[^4]: Auth.js database adapters. https://adapters.authjs.dev/
[^5]: Auth.js docs, edge compatibility and split-config (`auth.config.ts`) pattern. https://authjs.dev/guides/edge-compatibility

## Tags

typescript, authentication, oauth, oidc, nextjs, jwt, session-management, web-security, auth, library
