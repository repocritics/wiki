# ory/hydra

> A headless OAuth 2.0 and OpenID Connect server in Go that issues tokens but deliberately does not manage users — you bring the login screen.

[GitHub repo](https://github.com/ory/hydra) ·
[Official website](https://www.ory.com/docs/hydra) ·
[License: Apache-2.0](https://github.com/ory/hydra/blob/master/LICENSE)

## Overview

Ory Hydra is an OpenID Certified OAuth 2.0 authorization server and OpenID
Connect provider written in Go, first published in 2015[^1]. It implements the
token-issuing half of an identity system — authorization code, client
credentials, refresh, implicit and hybrid flows, token introspection and
revocation, dynamic client registration, JWKS management — and nothing else. It
is one of four servers in the Ory stack alongside Kratos (identity/user
management), Keto (Zanzibar-style permissions), and Oathkeeper (access proxy).

The defining design decision, and the source of most confusion for newcomers, is
that Hydra does **not** authenticate users. It has no password store, no
registration form, no session UI. When a browser hits an authorization
endpoint, Hydra redirects to a "login and consent app" that you build and host;
your app authenticates the user however it likes and calls back into Hydra's
admin API to accept or reject the request[^2]. This is the opposite of the
batteries-included model of Keycloak or Auth0, where user management and the
OAuth server ship as one product. The upside is total control over the login
experience and clean separation of concerns; the cost is that you cannot deploy
Hydra alone and get a working login — assembly is mandatory.

Hydra is genuinely used at scale — Ory cites OpenAI, Fandom, and others as
adopters, and reports the broader Ory stack handling billions of API requests
per day[^3]. The security-sensitive OAuth logic lives in a separate,
extensively reviewed library, ory/fosite, rather than in Hydra itself[^4].

## Getting Started

Run the server against an in-memory database for a first look (ports 4444 =
public API, 4445 = admin API):

```bash
docker run --rm -e DSN=memory -p 4444:4444 -p 4445:4445 \
  oryd/hydra:v2.2.0 serve all --dev
```

Create a client and run the client-credentials flow with the Ory CLI:

```bash
# Register an OAuth2 client against the ADMIN API (:4445)
hydra create oauth2-client \
  --endpoint http://127.0.0.1:4445 \
  --grant-type client_credentials \
  --format json

# Exchange credentials for a token against the PUBLIC API (:4444)
hydra perform client-credentials \
  --endpoint http://127.0.0.1:4444 \
  --client-id <id> --client-secret <secret>
```

For real deployments, point `DSN` at PostgreSQL, MySQL, or CockroachDB and run
`hydra migrate sql` before `hydra serve`. The `--dev` flag disables TLS
enforcement and must never be used in production.

## Architecture / How It Works

The heart of Hydra is the **login/consent redirect dance**. Nothing about it is
optional; it is how Hydra stays user-agnostic:

1. A client sends the user to Hydra's `/oauth2/auth` (public API).
2. Hydra generates a `login_challenge` and redirects the browser to the URL you
   configured as your login app.
3. Your app authenticates the user (session cookie, Kratos, LDAP, anything),
   then calls Hydra's admin API `PUT /admin/oauth2/auth/requests/login/accept`
   with the subject, and redirects back.
4. Hydra repeats the round trip for consent, emitting a `consent_challenge`; your
   app decides which scopes to grant and calls the consent accept endpoint.
5. Only then does Hydra mint the authorization code / tokens.

The **two-port split** is architectural, not cosmetic: the public API (4444)
serves the OAuth/OIDC endpoints exposed to the internet, while the admin API
(4445) — which can accept/reject login and consent, manage clients, and
introspect tokens — must never be publicly reachable. Exposing 4445 is a
critical misconfiguration.

Under the hood, the OAuth 2.0 and OIDC protocol handling is delegated to
**ory/fosite**, a standalone Go library that manages token entropy, PKCE,
constant-time secret comparison, and storage abstraction[^4]. Hydra is largely
the server, persistence, migrations, CLI, and admin surface wrapped around
fosite. Persistence uses a SQL schema managed by Hydra's own migration system;
supported stores are PostgreSQL, MySQL, and CockroachDB, with an in-memory mode
for tests only.

## Production Notes

**You must ship a login/consent app.** The reference implementation,
hydra-login-consent-node, is exactly that — a reference. It is not a
production-ready UI, and teams routinely underestimate the work of building,
securing, and maintaining this component. Pairing Hydra with Ory Kratos is the
sanctioned way to get real user management, but that is a second server to
operate.

**Run the janitor or the database will bloat.** Expired and rejected tokens,
consent/login requests, and flows accumulate in the store indefinitely. Hydra
ships `hydra janitor` for this; schedule it (cron, CronJob) or watch table sizes
grow without bound on high-throughput deployments.

**The 1.x → 2.0 upgrade is a real migration.** Hydra 2.0 (2022) reorganized the
CLI, SDKs, and admin API routes under `/admin`, and changed configuration and
client semantics; it is not a drop-in bump[^5]. Pin your version, read the
upgrade guide, and test the SQL migrations against a copy of production.

**CVE SLAs are a paid feature.** The open-source distribution receives security
fixes on Ory's schedule; guaranteed CVE patches with SLAs, current vetted
enterprise builds, and support for advanced multi-tenant scaling are gated
behind the commercial Ory Enterprise License[^6]. Ory positions self-hosted OSS
Hydra as suitable for experimentation and non-critical workloads. Budget for
either the license or your own security-monitoring discipline.

**Operational shape.** Hydra is stateless between requests (all state lives in
SQL), so it scales horizontally behind a load balancer cleanly. Keep the admin
API on a private network, terminate TLS in front, and treat the JWKS signing
keys as sensitive secrets with a rotation plan.

## When to Use / When Not

**Use when:**
- You need a standards-compliant, certified OAuth2/OIDC provider and want to own
  the login UI and user store yourself.
- You already run (or will run) Ory Kratos or another identity source and want a
  clean, separately-scaled token server.
- You want cloud-native, horizontally-scalable OAuth in Go with a permissive
  Apache-2.0 license.

**Avoid when:**
- You want an out-of-the-box login page, user database, and admin console with
  minimal assembly — reach for Keycloak or Zitadel.
- You only need to federate to existing upstream IdPs (GitHub, LDAP, SAML) — a
  broker like Dex is lighter.
- Your team cannot commit to building and securing a login/consent app, or to
  the security-patching discipline the OSS build requires.

## Alternatives

- keycloak/keycloak — full IAM with bundled user management, admin UI, and
  federation; use instead when you want batteries included and don't want to
  build login/consent yourself.
- zitadel/zitadel — cloud-native IAM in Go with users, multi-tenancy, and OIDC
  in one binary; use when you want a Hydra-like stack but with identity bundled.
- dexidp/dex — lightweight OIDC provider that federates to upstream connectors;
  use when you mainly need an OIDC broker in front of existing IdPs.
- casdoor/casdoor — UI-first, self-hosted IAM/SSO; use when a ready-made admin
  and login interface matters more than decoupling.
- ory/kratos — not a replacement but the intended companion; add it when you
  need the user management that Hydra deliberately omits.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-05 | First public commit of the OAuth2/OIDC server[^1]. |
| 1.0.0 | 2019 | First stable GA; used for OpenID certification[^7]. |
| 2.0.0 | 2022-11 | Major release: reworked CLI/SDK, admin API under `/admin`[^5]. |
| 2.x | 2023–2026 | Ongoing 2.x line; Ory Network managed offering, OEL enterprise builds. |

## References

[^1]: ory/hydra repository, created 2015-05-22. https://github.com/ory/hydra
[^2]: Ory docs, "Login and consent flow." https://www.ory.com/docs/oauth2-oidc/custom-login-consent/flow
[^3]: Ory Hydra README, "Who is using Ory Hydra" (adopters incl. OpenAI, Fandom). https://github.com/ory/hydra
[^4]: ory/fosite — security-first OAuth2 SDK underpinning Hydra. https://github.com/ory/fosite
[^5]: Ory Hydra v2.0 upgrade guide. https://www.ory.com/docs/hydra/self-hosted/upgrade
[^6]: Ory Enterprise License. https://www.ory.com/ory-enterprise-license
[^7]: OpenID Foundation certified OpenID Providers. https://openid.net/certification/

## Tags

go, oauth2, openid-connect, oidc, authorization-server, identity, security, sso, self-hosted, ory-stack, headless
