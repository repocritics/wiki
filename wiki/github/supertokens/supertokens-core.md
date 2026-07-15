# supertokens/supertokens-core

> The Java HTTP service at the center of SuperTokens — a self-hostable, open-core alternative to Auth0, Firebase Auth, and AWS Cognito.

[GitHub repo](https://github.com/supertokens/supertokens-core) ·
[Official website](https://supertokens.com) ·
License: Apache-2.0 (core; the `ee/` directory ships under a separate SuperTokens Enterprise license)[^1]

## Overview

SuperTokens is a user-authentication stack split across three layers: a Frontend SDK (renders login widgets, holds session tokens), a Backend SDK (exposes sign-up / sign-in / refresh / sign-out APIs your server mounts), and the **Core** — this repo — a stateless Java service that owns the auth logic and talks to your database[^2]. The pitch is that you self-host all three, keep 100% of user data in your own Postgres or MySQL, and never pay per monthly-active-user. A managed cloud is offered, but the free self-hosted path has no user cap.

The defining architectural choice is that the Core is *not* on your app's hot path. The most frequent auth operation — verifying a session on each request — happens inside the Backend SDK by validating a JWT against a cached JWKS, without an HTTP round-trip to the Core[^3]. The Core is only contacted for state-changing operations (login, refresh, user CRUD). This is what lets a single Java instance back tens of thousands of users, and it is the reason "why Java" (a heavier runtime than the SDK languages) is defensible: the Core is infrastructure you run a couple replicas of, not a per-request dependency.

The tradeoff of the three-layer design is integration surface. You are wiring together a Core deployment, a database plugin, a Backend SDK pinned to a compatible Core version, and a Frontend SDK — four moving parts that must agree on versions and CORS/cookie domains. Compared to a single library like Auth.js, that is more to stand up; compared to a hosted IdP, it is more to operate. What you buy is data ownership and no MAU billing.

## Getting Started

Run the Core (bundled JDK, embedded Tomcat) with the managed-DB-free dev image:

```bash
# Core listening on :3567. Use the -postgresql or -mysql image in production.
docker run -p 3567:3567 -d registry.supertokens.io/supertokens/supertokens-postgresql
```

Mount the Backend SDK (Node.js shown) and point it at the Core:

```js
import supertokens from "supertokens-node";
import Session from "supertokens-node/recipe/session";
import EmailPassword from "supertokens-node/recipe/emailpassword";

supertokens.init({
  supertokens: { connectionURI: "http://localhost:3567" }, // the Core
  appInfo: {
    appName: "MyApp",
    apiDomain: "http://localhost:3001",
    websiteDomain: "http://localhost:3000",
  },
  recipeList: [EmailPassword.init(), Session.init()],
});
```

The SDK auto-mounts routes like `/auth/signup` and `/auth/session/refresh`; the Frontend SDK calls them.

## Architecture / How It Works

The Core is a plain HTTP service, not a framework you embed. Its internals:

- **Database plugin layer.** The Core defines a storage interface; the concrete driver is a separate plugin JAR (`supertokens-postgresql-plugin`, `supertokens-mysql-plugin`). An in-memory store exists for tests/dev only — it loses all state on restart, which surprises people who try to demo with it.
- **Recipes.** Auth methods are modular: EmailPassword, ThirdParty (social/OAuth login), Passwordless (magic link / OTP), Session, UserRoles, and — in `ee/` — Multi-Tenancy, MFA, and account linking. You enable only the recipes you use; unused recipes add no endpoints.
- **CDI (Core Driver Interface).** The Core and Backend SDK negotiate a versioned contract. A given SDK release supports a range of Core versions; mismatches fail at startup with a version error. This coupling is the most common upgrade friction — you cannot bump the Core arbitrarily ahead of the SDK.
- **Session model.** Sessions issue a short-lived access token (a JWT the SDK verifies locally) plus a rotating refresh token. Refresh-token rotation with theft detection is the default: a stolen-and-reused refresh token invalidates the session family. This is stronger than a bare long-lived JWT but means the Core (and DB) must be reachable for refresh.

Because the Core is stateless, horizontal scaling is "run N replicas behind a load balancer, all pointed at the same database." There is no inter-node coordination; the database is the single source of truth. That also makes the database the scaling ceiling.

## Production Notes

**The `ee/` split is a real licensing boundary, not a paywall banner.** Enterprise features (multi-tenancy, some SSO/SAML, account linking) live under `ee/` and are governed by the SuperTokens Enterprise license — using them in production requires a license key even though the source is in the repo[^1]. Read `ee/LICENSE.md` before you build a feature on top of a directory you didn't check.

**Version-lock the trio.** Core, Backend SDK, and Frontend SDK versions are interdependent via CDI/FDI. Upgrades are not "bump one and go" — the docs publish a compatibility matrix, and skipping it produces startup version errors or subtle session-format breaks. Pin all three and upgrade in lockstep.

**In-memory DB is a dev trap.** The default `supertokens/supertokens` image (no `-postgresql`/`-mysql` suffix) uses the in-memory store; restarts wipe users. For anything real, use the DB-specific image and provide `POSTGRESQL_CONNECTION_URI` / equivalent.

**Cookies and domains.** Session tokens default to HttpOnly cookies, which means `apiDomain`/`websiteDomain` and `cookieDomain` must be correct across subdomains, and cross-site setups need `sameSite: "none"` + HTTPS. Misconfigured domains are the top source of "login works locally, sessions drop in prod" reports.

**JVM footprint.** It is a Java service; expect a few hundred MB RSS per replica at idle — heavier than a Go or Rust IdP. The maintainers have discussed GraalVM native images to cut this, but plan for JVM-sized containers today.

**Self-hosting is the supported path, but you own DB backups, migrations, and TLS.** Core upgrades can carry DB migrations; run them in a maintenance step and back up first.

## When to Use / When Not

**Use when:**
- You want data residency / no per-MAU billing and are willing to run a service + database.
- You need pre-built login (email/password, social, passwordless) wired into your own backend rather than a redirect-to-hosted-IdP flow.
- You want session security (refresh-token rotation, theft detection) without hand-rolling it.
- Your stack has a supported Backend SDK (Node, Python, Go) and a supported framework integration.

**Avoid when:**
- You want zero infrastructure — a hosted IdP (Auth0, Clerk, Cognito) or an in-process library (Auth.js) is less to operate.
- Your backend language has no first-party SDK; you'd be talking to the Core's HTTP API by hand.
- You need full OIDC/OAuth2 *provider* capabilities out of the box (issuing tokens to third-party clients) — that is Keycloak/Ory/ZITADEL's home turf more than SuperTokens'.
- You want a single small binary — the JVM footprint and multi-component setup are heavier than Go-based alternatives.

## Alternatives

- keycloak/keycloak — full-featured OIDC/SAML IdP; use when you need standards-compliant federation and admin console over data-ownership simplicity.
- ory/kratos — headless identity API (Go); use when you want to build the entire UI yourself and prefer a spec-driven, API-first identity server.
- zitadel/zitadel — Go IdP with multi-tenancy and OIDC built in; use when you want cloud-or-self-host with a managed-service parity story.
- goauthentik/authentik — Python IdP/proxy; use when you want an application-gateway/SSO portal in front of many apps.
- nextauthjs/next-auth (Auth.js) — in-process JS library, no separate service; use when a single Next.js/JS app just needs login and you don't want to run infrastructure.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-01 | Repo created; SuperTokens launches as a self-hosted session-management + auth service[^4]. |
| — | 2020–2022 | Recipe model matures: EmailPassword, ThirdParty, Passwordless, Session. |
| — | 2022–2023 | Multi-tenancy, MFA, and account linking added under the `ee/` enterprise license[^1]. |
| ongoing | 2026 | Actively developed; latest push 2026-07-14. ~15.3k stars, ~830 forks. |

## References

[^1]: SuperTokens `LICENSE.md` — Apache-2.0 for the core, with content under `ee/` governed by a separate SuperTokens Enterprise license. https://github.com/supertokens/supertokens-core/blob/master/LICENSE.md
[^2]: SuperTokens README, "Three building blocks of SuperTokens architecture" (Frontend SDK, Backend SDK, Core). https://github.com/supertokens/supertokens-core
[^3]: SuperTokens README, "Why Java" — session verification happens in the Backend SDK without contacting the Java Core. https://github.com/supertokens/supertokens-core#-why-java
[^4]: GitHub repository metadata — `created_at` 2020-01-05. https://github.com/supertokens/supertokens-core

## Tags

authentication, session-management, self-hosted, java, open-core, auth-provider, oauth, passwordless, social-login, jwt, identity, docker
