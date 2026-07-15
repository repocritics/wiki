# authlib/authlib

> Framework-agnostic Python implementation of OAuth 1.0, OAuth 2.0, OpenID Connect, and the JOSE stack — as both client and server building blocks.

[GitHub repo](https://github.com/authlib/authlib) ·
[Official website](https://authlib.org) ·
[License: BSD-3-Clause](https://github.com/authlib/authlib/blob/main/LICENSE)

## Overview

Authlib is a Python library for building OAuth and OpenID Connect *clients and
providers*, plus a from-scratch JOSE implementation (JWS, JWE, JWK, JWA, JWT). It
is written and maintained primarily by Hsiaoming Yang (`lepture`), and the repo
now lives under the `authlib` GitHub org after moving from `lepture/authlib`[^1].
The scope is unusually wide for one project: it implements roughly a dozen OAuth
2.0 RFCs (6749, 6750, 7009, 7523, 7591, 7636, 7662, 8414, 8628, 9068, 9101,
9207) and the OIDC core/discovery/registration specs, then layers framework
integrations for requests, HTTPX, Flask, Django, Starlette, and FastAPI[^2].

The defining tension is that Authlib is a *toolkit*, not a drop-in auth product.
The client integrations are close to batteries-included, but building a provider
means implementing a set of storage and lifecycle hooks yourself
(`query_client`, `save_authorization_code`, `save_token`, grant classes, token
generators). You get spec correctness and control; you supply the persistence,
UI, and policy. Teams that expect a Keycloak-in-a-pip-install are consistently
surprised by how much wiring the server side requires.

The second tension is governance and funding. Authlib is BSD-licensed but is a
single-maintainer project offered under a dual model — the open BSD library plus
a paid commercial license and a "fund Authlib to access additional features"
sponsorship track[^3]. That funding model is legitimate and keeps the lights on,
but it means bus factor and roadmap both concentrate on one person.

## Getting Started

```bash
pip install authlib
# server-side JWT/JOSE work also commonly pulls in cryptography (a dependency)
```

```python
# Verifying a JWT with a JWK set — client side
from authlib.jose import jwt

# `key` may be a JWK dict, a PEM, or a JWKSet fetched from a provider's JWKS URL
claims = jwt.decode(token, key)
claims.validate()          # checks exp/nbf/iat and any registered claim options
print(claims["sub"])
```

```python
# OAuth 2.0 client with the requests integration
from authlib.integrations.requests_client import OAuth2Session

sess = OAuth2Session(client_id, client_secret, scope="openid profile")
uri, state = sess.create_authorization_url("https://provider/authorize")
# ... redirect the user, receive the callback ...
token = sess.fetch_token("https://provider/token", authorization_response=cb_url)
```

## Architecture / How It Works

Authlib is layered deliberately. At the bottom sit framework-agnostic RFC
implementations: the JOSE package, and OAuth grant/endpoint classes that operate
on plain request/response abstractions. On top of those sit thin per-framework
integration packages under `authlib.integrations.*` that adapt Flask, Django,
Starlette, FastAPI, requests, or HTTPX request objects to the core.

For **providers**, you compose an `AuthorizationServer` and register grant
classes (authorization code, client credentials, refresh token, device code,
etc.). Each grant calls into hooks you implement against your own models — there
is no bundled ORM or schema. PKCE, token introspection, revocation, and dynamic
client registration are separate RFC modules you opt into. OIDC support is built
as an extension grant on top of the OAuth 2.0 code grant, adding `id_token`
minting via the JOSE layer.

For **clients**, `OAuth2Session` / `AsyncOAuth2Client` subclass the underlying
requests / HTTPX session so that token attachment, refresh, and OAuth signing
happen transparently on normal HTTP calls. The web-framework client integrations
(`authlib.integrations.flask_client`, `django_client`, `starlette_client`) add
session-backed state, nonce handling, and a registry for named remote providers
(the common "log in with Google/GitHub" pattern).

The **JOSE** layer is a notable sub-project of its own. It implements the
algorithms directly rather than wrapping a single crypto library's JWT helper,
which is what lets Authlib cover JWE and the full JWK/JWA matrix. That same layer
is now being spun out: `authlib.jose` is deprecated in favor of a standalone
library, `joserfc`, also by the same author[^4].

## Production Notes

- **`authlib.jose` is deprecated.** New code should target `joserfc`; the README
  ships a migration guide[^4]. Existing `authlib.jose` imports still work but are
  on a sunset path, so pin versions and plan the migration rather than being
  surprised by a future removal.
- **Algorithm allow-lists matter.** JWT verification is only as safe as the
  algorithms you permit. Do not decode with an open-ended algorithm set; pass an
  explicit list so a token cannot downgrade to `none` or coerce an
  RSA-public-key-as-HMAC-secret confusion. This is a JOSE-wide footgun, not
  unique to Authlib, but Authlib gives you enough rope to get it wrong.
- **Providers are DIY persistence.** There is no default database layer. You own
  client storage, token storage, code expiry, and cleanup of expired tokens.
  Getting refresh-token rotation and revocation right is on you.
- **Single-maintainer bus factor.** Development, security triage, and releases
  route through one person (security reports go to a personal email / Tidelift
  coordination)[^5]. Response time is generally good but concentrated.
- **Funding-gated features.** Some capabilities sit behind the commercial license
  or the funding tier[^3]. Read the docs' feature/funding notes before assuming
  a spec is covered in the free build.
- **Version pinning across the 1.x line.** Behavior around JOSE claim validation
  and grant registration has shifted between minor releases; pin an exact version
  in provider deployments and read changelogs before bumping.

## When to Use / When Not

**Use when:**
- You need spec-correct OAuth 2.0 / OIDC *server* pieces embedded in a Flask,
  Django, or Starlette app and are willing to wire storage yourself.
- You want one library that covers both the client and provider sides plus JOSE.
- You need less-common RFCs (device grant, JAR, token exchange, DCR) that
  client-only libraries omit.

**Avoid when:**
- You want a turnkey identity product with a UI, user store, and admin — run a
  server (Keycloak, Ory Hydra, Zitadel) instead of building one.
- You only need to sign/verify JWTs — a focused library is simpler and the JOSE
  module is being migrated out anyway.
- Single-maintainer bus factor or feature-gating behind funding is a procurement
  blocker for your org.

## Alternatives

- oauthlib/oauthlib — lower-level, spec-only OAuth logic; powers requests-oauthlib. Use when you want a minimal core and will build the rest.
- requests/requests-oauthlib — client-side OAuth only. Use when you never need to *be* the provider.
- jpadilla/pyjwt — JWT encode/decode only. Use when your entire need is signing and verifying tokens.
- jazzband/django-oauth-toolkit — opinionated Django OAuth2 provider with models included. Use when you're on Django and want batteries, not building blocks.
- ory/hydra — standalone OAuth2/OIDC server as a service. Use when you want to run an identity server, not embed a library.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017–2021 | Initial OAuth 1/2 + JOSE implementation; rapid RFC coverage under `lepture/authlib`. |
| 1.0.0 | 2022-02 | First stable major; API cleanup, dropped legacy Python. |
| 1.2.x | 2023 | Additional RFC modules and framework-integration refinements. |
| 1.3.x | 2024 | Continued OAuth2/OIDC and integration updates. |
| — | 2024→ | `joserfc` split out; `authlib.jose` marked for deprecation[^4]. |

*(Exact patch-level dates vary; consult the repo's release notes and CHANGELOG
for precise versions before pinning.)*

## References

[^1]: Repository metadata — `authlib/authlib`, BSD-3-Clause, ~5.4k stars, created 2017-10-27, last pushed 2026-06 (GitHub API, redirected from `lepture/authlib`).
[^2]: Authlib README — feature/RFC list and framework integrations. https://github.com/authlib/authlib
[^3]: Authlib licensing and funding — BSD plus commercial license and funding tier. https://authlib.org/plans and https://docs.authlib.org/en/stable/community/funding.html
[^4]: "Migrating from `authlib.jose` to `joserfc`." https://jose.authlib.org/en/dev/migrations/authlib/
[^5]: Authlib security reporting policy (private disclosure via maintainer email / Tidelift). https://github.com/authlib/authlib#security-reporting

## Tags

python, oauth2, oauth1, openid-connect, oidc, jwt, jose, jws, jwe, jwk, authentication, flask
