# casdoor/casdoor

> A self-hosted, UI-first identity provider and SSO server in Go, from the Casbin team — one console for OAuth2/OIDC, SAML, CAS, LDAP, SCIM, and MFA.

[GitHub repo](https://github.com/casdoor/casdoor) ·
[Official website](https://casdoor.ai) ·
[License: Apache-2.0](https://github.com/casdoor/casdoor/blob/master/LICENSE)

## Overview

Casdoor is an open-source identity and access management (IAM) / single-sign-on server built by the maintainers of Casbin, the Go authorization library[^1]. Where Casbin answers "is this subject allowed to do this action," Casdoor is the layer above it: it authenticates users and issues tokens, exposing a web console to manage organizations, users, applications, and identity providers. It first appeared in the Casbin ecosystem around 2020[^2] and has since become one of the more feature-broad self-hostable identity servers in Go.

Its defining trait is protocol breadth behind an admin UI. A single deployment can act as an OAuth 2.0 / OIDC provider, a SAML 2.0 IdP, a CAS server, an LDAP server *and* client, a SCIM provisioning endpoint, and a WebAuthn/TOTP MFA backend — all configured by clicking through forms rather than editing config files or writing Terraform. That is the appeal for teams who want Keycloak-style coverage without the JVM, and the risk: the surface area is large, the release cadence is very fast, and the depth of any single protocol is not always as battle-tested as a specialist tool.

A second thing to understand before adopting: as of 2025–2026 the project has repositioned around AI, describing itself as an "agent-first" IAM and MCP / AI gateway[^3]. The MCP-gateway, A2A, and "AI-first" framing is genuinely new code, but it is layered onto a mature IAM core — evaluate the identity fundamentals on their own merits and treat the agent-gateway features as early-stage.

## Getting Started

Docker is the fastest path to a working instance (SQLite, for trials only):

```bash
docker run -p 8000:8000 casbin/casdoor-all-in-one
# open http://localhost:8000  →  login: built-in/admin  password: 123
```

From source (Go 1.25+, Node.js 20, Yarn 1.x, and a real database for anything beyond a demo):

```bash
git clone https://github.com/casdoor/casdoor.git
cd casdoor
# set driverName / dataSourceName / dbName in conf/app.conf
cd web && yarn install && yarn build && cd ..
go run main.go
```

Registering an OIDC client is done in the UI (Applications → new), then consumed like any OIDC provider:

```
Authorization endpoint:  https://<host>/login/oauth/authorize
Token endpoint:          https://<host>/api/login/oauth/access_token
OIDC discovery:          https://<host>/.well-known/openid-configuration
```

Change the `built-in/admin` / `123` credentials before exposing anything to a network.

## Architecture / How It Works

Casdoor is a frontend–backend-separated application:

- **Backend** — Go on the **Beego** web framework, exposing a RESTful API. Persistence goes through an ORM layer (XORM) that auto-syncs the schema against mainstream SQL databases (MySQL, PostgreSQL, SQL Server, SQLite, and others). Casbin itself is embedded to enforce access control on Casdoor's own API resources.
- **Frontend** — a React single-page app (Ant Design components) under `web/`, which can be built and embedded into the Go binary or served separately.

The data model is the thing to internalize, because every feature hangs off it:

- **Organization** — the top-level tenant boundary. Every user, application, and provider belongs to one. A `built-in` organization ships by default.
- **User** — the identity, scoped to an organization.
- **Application** — an OAuth/OIDC/SAML client definition, with its own signup/login page config and allowed providers.
- **Provider** — an upstream or downstream integration: social login (Google, GitHub, Azure AD), SMS/email senders, storage, payment, captcha, or MFA backends.
- **Cert / Token** — signing certificates and issued tokens.
- **Role / Permission** — RBAC objects that map onto Casbin policies.

Authentication protocols are implemented as endpoints on top of this model: the OIDC endpoints, a SAML IdP, a CAS `/serviceValidate`, an LDAP server so legacy apps can bind against Casdoor, and SCIM for user provisioning. Because the model is shared, a user created via the UI is immediately usable across every protocol — that unification is the core design value.

## Production Notes

**Release cadence is continuous, not semantic.** Casdoor ships extremely frequently — minor-version tags land multiple times per week (v3.117.0 was tagged in mid-July 2026, following v3.116.0 the same day)[^4]. The `v3.x` numbering does not signal API-stability tiers; there is no LTS line. Pin an exact image tag, read release notes before bumping, and do not track `latest` in production. The upside is fast fixes; the cost is that you are effectively always on a young release.

**Schema auto-sync is a double-edged tool.** The ORM synchronizes tables on startup. This makes first-run painless but means an upgrade can alter your schema implicitly — snapshot the database before version bumps, and do not point two different Casdoor versions at the same database.

**Configuration is Beego-style `conf/app.conf`.** Database DSN, session settings, and Redis (optional, for session/cache) live there; some values are also overridable by environment variables. There is no first-class secrets story — inject `dataSourceName` and provider secrets via your own mechanism.

**Multi-tenancy is organization-scoped, single-instance.** Organizations partition users and apps within one Casdoor deployment; this is not the same as running isolated instances per tenant. Understand the `built-in` org's privileged status before you design around it.

**Documentation moved and can lag.** Docs now live at `casdoor.ai` (previously `casdoor.org`); given the release speed, docs and UI occasionally drift from the newest features. The live read-only demo (`door.casdoor.com`) and its Swagger explorer are useful for checking actual API shapes.

**AI-gateway features are the newest and least proven.** The MCP gateway / A2A / agent-auth surfaces are recent additions. If you are adopting Casdoor, weight your evaluation toward the IAM core (OIDC/SAML/LDAP), which has years of production use, and treat the agent-gateway layer as evolving.

## When to Use / When Not

**Use when:**
- You want one self-hosted server covering OIDC + SAML + CAS + LDAP + SCIM + MFA, administered from a web UI.
- You prefer a Go single-binary-ish deployment over a JVM stack.
- You are already in the Casbin ecosystem and want a matching identity layer.
- You need to stand up SSO quickly for internal apps and value click-to-configure over declarative config.

**Avoid when:**
- You need a slow, predictable, LTS-style upgrade path — the continuous release model works against you.
- You want an API-first, headless identity system with no opinionated UI (Ory's split-service model fits better).
- You need deep, certified conformance in one protocol (a specialist SAML or OIDC product may be safer than a broad generalist).
- Your requirement is reverse-proxy request authentication rather than a full IdP (a proxy-companion tool is a better shape).

## Alternatives

- keycloak/keycloak — the mature enterprise reference IdP; far broader hardening and ecosystem, at the cost of a heavier Java/Quarkus footprint. Use when JVM operational maturity matters more than a lightweight binary.
- zitadel/zitadel — Go, modern, strongly multi-tenant with an event-sourced core. Use when tenant isolation and audit-grade event history are central.
- ory/kratos — API-first, headless identity (pair with ory/hydra for OAuth2). Use when you want to own the UI and prefer composable services over an admin console.
- authelia/authelia — lightweight authentication companion for reverse proxies. Use when you need to gate apps behind an existing proxy, not run a full IdP.
- logto-io/logto — developer-experience-focused OIDC provider with prebuilt flows. Use when DX and a clean B2C/B2B onboarding path outweigh protocol breadth.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2020-10-22 | Repository created under the Casbin org[^2]. |
| v3.35.x | 2026-04 | Continuous `v3.x` line; releases tagged several times per week[^4]. |
| — | 2025–2026 | Repositioning around AI: MCP gateway, A2A, "agent-first" IAM framing[^3]. |
| v3.117.0 | 2026-07-14 | Latest tag at time of writing; ~14k stars, 1.7k forks[^5]. |

## References

[^1]: Casbin — authorization library for Go and other languages, by the same organization. https://casbin.org/
[^2]: casdoor/casdoor repository metadata (created 2020-10-22), GitHub API. https://github.com/casdoor/casdoor
[^3]: Casdoor README and repository description, "AI-first IAM / MCP gateway". https://github.com/casdoor/casdoor
[^4]: casdoor/casdoor releases (v3.116.0 and v3.117.0 both tagged 2026-07-14), GitHub API. https://github.com/casdoor/casdoor/releases
[^5]: casdoor/casdoor GitHub API stats, retrieved 2026-07-15: 13,937 stars, 1,737 forks, Apache-2.0.

## Tags

go, iam, sso, authentication, oauth2, oidc, saml, ldap, identity-provider, self-hosted, casbin, mcp-gateway
