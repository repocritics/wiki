# ory/kratos

> Headless, API-first identity and user management in Go — you own the UI, Kratos owns the flows.

[GitHub repo](https://github.com/ory/kratos) ·
[Official website](https://www.ory.com/kratos) ·
[License: Apache-2.0](https://github.com/ory/kratos/blob/master/LICENSE)

## Overview

Ory Kratos is an identity and user management server for cloud-native applications, written in Go and first opened publicly around 2020[^1]. It handles the workflows almost every app reimplements badly: self-service registration, login, account recovery, email/SMS verification, multi-factor auth, and profile/settings management. The defining design choice is that it is *headless* — Kratos ships no login pages. It exposes flows as JSON over HTTP and returns a description of the form fields ("UI nodes") to render; you build the actual screens in whatever frontend you already have.

That headlessness is the central tradeoff. It buys near-total control over UX, branding, and framework choice, and it keeps identity logic out of your application code. It costs you a real integration project: you must render flows, follow redirects, manage the session cookie/token, and wire a reference UI before anything is usable. Teams expecting a drop-in login page are routinely surprised.

The second thing to understand is that Kratos is one component of the Ory stack, not the whole thing. Kratos does *not* speak OAuth2 or OpenID Connect as a provider — that is Ory Hydra's job. Kratos handles the human-facing credential and profile side; Hydra issues tokens; Keto does Zanzibar-style permissions; Oathkeeper is the access proxy[^2]. Replacing an Auth0/Okta that your apps consume over OIDC therefore usually means running Kratos *and* Hydra together, not Kratos alone.

## Getting Started

```bash
# Self-host: point at a database, apply migrations, then serve.
export DSN="postgres://kratos:secret@localhost:5432/kratos?sslmode=disable"
kratos migrate sql -e --yes
kratos serve --config ./kratos.yml --dev
```

Identities are defined by a JSON Schema you supply, which maps traits to credentials:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "traits": {
      "type": "object",
      "properties": {
        "email": {
          "type": "string",
          "format": "email",
          "ory.sh/kratos": {
            "credentials": { "password": { "identifier": true } },
            "verification": { "via": "email" }
          }
        }
      },
      "required": ["email"]
    }
  }
}
```

Because Kratos is headless, a flow is initialized server-side and returns the fields to render:

```bash
# Public API on :4433, Admin API on :4434.
curl -s -H "Accept: application/json" \
  http://127.0.0.1:4433/self-service/login/browser | jq '.ui.nodes'
```

## Architecture / How It Works

Kratos is organized around **self-service flows** — one per user intent: `login`, `registration`, `recovery`, `verification`, `settings`. Each flow has two entry styles that behave differently[^3]:

- **Browser flows** — for server-rendered or SPA web apps. They rely on redirects, anti-CSRF cookies, and a session cookie. The browser is expected to follow 303 redirects between Kratos and your UI.
- **API flows** — for native/mobile clients. No cookies or redirects; the client drives the flow explicitly and receives a session token.

A flow is stateful: you initialize it, Kratos returns UI nodes plus a flow ID, the user submits, and Kratos either advances the flow or returns validation messages attached to specific nodes. Your UI is a thin renderer of that node list, which is why Ory ships reference frontends (self-service-ui-node, Ory Elements) rather than expecting you to hand-roll every field.

**Identities and traits.** There is no fixed user table shape. You define an identity schema (JSON Schema); the `ory.sh/kratos` extension marks which trait is the login identifier, what gets verified, and how it maps to credential types (password, OIDC/social, WebAuthn/passkey, TOTP, lookup codes, code-via-email). Changing the schema is a migration, not a config toggle.

**Sessions are opaque, not JWTs.** After login Kratos issues a session cookie (browser) or token (API), and services validate it by calling `/sessions/whoami`. There is an optional feature to mint a JWT from a session for downstream consumers, but the source of truth is server-side, revocable session state — a deliberate contrast to stateless-JWT designs[^4].

**Courier.** Email and SMS (verification, recovery, MFA codes) are queued in the database and delivered by a courier worker. This is a separate concern from serving HTTP and a frequent source of "why didn't my email send" confusion (see below).

**Storage** is SQL — PostgreSQL, MySQL, CockroachDB, or SQLite for local/dev. Configuration is a single YAML file plus the identity schema(s), which makes the deployment GitOps-friendly but means most behavior changes are config edits and restarts, not admin-console clicks.

## Production Notes

- **The courier is a separate process.** Serving the API does not send mail. You must run `kratos courier watch` (or enable the background courier) or verification/recovery messages silently pile up in the DB. This is the single most common self-host footgun.
- **`whoami` latency.** Every session check is a network call to Kratos. High-traffic gateways should cache session lookups or use Oathkeeper; naive per-request `whoami` calls can dominate request latency.
- **Migrations on every upgrade.** `kratos migrate sql` must run before serving a new version. Rolling deploys need care: run migrations as a gated step, and read the release notes — some minor versions carry non-trivial schema changes.
- **Open-source vs Enterprise split.** SAML, SCIM, organization SSO ("multi-org" login), and CAPTCHA are **not** in the Apache-2.0 build; they require the Ory Enterprise License and a private Docker registry[^5]. Teams that assumed enterprise SSO was included have had to re-plan. Confirm feature availability against the OSS distribution before committing.
- **Config surface is large.** Flow methods, hooks (e.g. run a webhook after registration), redirect URLs, cookie domains, and CORS all live in YAML. Misconfigured cookie domains and CORS between the Kratos public URL and your UI origin are the usual cause of "flow works locally, breaks in prod."
- **Not an OAuth2 server.** If you need to issue access tokens to third-party clients or do machine-to-machine auth, that is Hydra. Reaching for Kratos alone here is a common architectural mismatch.

## When to Use / When Not

**Use when:**
- You want full control of the login/registration UI and are willing to build it.
- You need identity flows decoupled from your app code, in a self-hostable Go binary with a SQL backend.
- You already run (or will run) other Ory components and want a coherent, API-compatible stack.
- You want revocable, server-side sessions and passkeys/WebAuthn/TOTP without building credential handling yourself.

**Avoid when:**
- You want a batteries-included IdP with a prebuilt admin console and hosted login pages — Keycloak, Zitadel, or authentik fit better.
- You need SAML/SCIM/org-SSO and can't take the Ory Enterprise License.
- Your primary need is being an OAuth2/OIDC *provider* — that's Hydra, not Kratos.
- The team can't budget an integration project to render flows and wire a UI.

## Alternatives

- keycloak/keycloak — use instead when you want a mature, all-in-one IdP with built-in admin UI, hosted login themes, and SAML/OIDC out of the box (JVM, heavier).
- zitadel/zitadel — use when you want cloud-native Go like Kratos but batteries-included: OIDC provider, UI, and multi-tenancy in one binary.
- goauthentik/authentik — use when you want a self-hosted IdP with a UI covering OIDC, SAML, and LDAP for internal/SSO scenarios.
- supertokens/supertokens-core — use when you want open-source auth with prebuilt frontend SDKs and less flow-wiring than Kratos.
- logto/logto — use when you want a modern, developer-friendly OSS auth stack with prebuilt sign-in UI and OIDC included.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2018-05-29 | Kratos development begins under the Ory org[^6]. |
| 0.x | 2020 | First public releases; self-service flow model and identity-schema design established[^1]. |
| 1.0.0 | 2023-08 | First stable major; API stability guarantees for the OSS server[^7]. |
| 1.x | 2023–2026 | Passkeys/WebAuthn, code-based login, ongoing hardening; enterprise-only features (SAML, SCIM, org SSO) delivered via OEL[^5]. |

## References

[^1]: Ory Kratos documentation — introduction and concepts. https://www.ory.com/kratos/docs/
[^2]: Ory stack overview — Kratos (identity), Hydra (OAuth2/OIDC), Keto (permissions), Oathkeeper (access proxy). https://www.ory.com/docs/
[^3]: Ory Kratos — self-service flows (browser vs API flows). https://www.ory.com/docs/kratos/self-service
[^4]: Ory Kratos — sessions and `/sessions/whoami`. https://www.ory.com/docs/kratos/session-management/overview
[^5]: Ory Enterprise License — enterprise features (SCIM, SAML, org SSO, CAPTCHA) and support. https://www.ory.com/ory-enterprise-license
[^6]: GitHub API — repository metadata for ory/kratos (created 2018-05-29). https://github.com/ory/kratos
[^7]: Ory Kratos releases (v1.0.0). https://github.com/ory/kratos/releases

## Tags

go, identity-management, authentication, headless, self-hosted, oidc, user-management, cloud-native, passkeys, session-management, ory-stack
