# jpadilla/pyjwt

> A minimal Python implementation of JSON Web Tokens (RFC 7519) — signing and verification only, no JWE, no full JOSE.

[GitHub repo](https://github.com/jpadilla/pyjwt) ·
[Official website](https://pyjwt.readthedocs.io) ·
[License: MIT](https://github.com/jpadilla/pyjwt/blob/master/LICENSE)

## Overview

PyJWT is the de facto standard library for encoding and decoding JSON Web Tokens in Python. It implements RFC 7519[^1] — the JWT profile of the JOSE family — and the JWS (signed token) portions of RFC 7515. The original implementation was written by Jeff Lindsay (@progrium); it has been maintained by José Padilla (@jpadilla) for most of its life. It is small, dependency-light, and used transitively by a large fraction of Python web auth: `djangorestframework-simplejwt`, most OIDC/Auth0/Cognito client code, and countless hand-rolled session systems depend on it.

The scope is deliberately narrow: PyJWT signs and verifies tokens, it does not encrypt them. A JWT payload is base64url-encoded, not confidential — anyone holding the token can read the claims. There is no JWE (encrypted token) support and no general JOSE toolkit; PyJWT does one layer of the standard and stops. This narrowness is the reason it is easy to audit and hard to misuse catastrophically — but it also means teams needing encryption, key wrapping, or full JWK set management reach for `jwcrypto` or `python-jose` instead.

The defining tension in PyJWT's history is **algorithm-choice safety**. JWTs carry their own `alg` header, and naive verifiers that trust that header are vulnerable to well-known attacks (the `none` algorithm, and RS256→HS256 confusion where an attacker signs a token with the RSA public key treated as an HMAC secret). PyJWT has progressively closed these holes by forcing the caller to be explicit, culminating in the 2.0 requirement that `decode()` always receive an `algorithms=` allowlist.

## Getting Started

```bash
pip install pyjwt
# for RSA / ECDSA / EdDSA / PSS you also need the crypto backend:
pip install "pyjwt[crypto]"
```

```python
import jwt

# HMAC (symmetric) — works with the standard library, no extra deps
token = jwt.encode({"user_id": 42}, "secret", algorithm="HS256")
payload = jwt.decode(token, "secret", algorithms=["HS256"])
# {'user_id': 42}

# Registered claims are validated automatically when present
import datetime
token = jwt.encode(
    {"user_id": 42, "exp": datetime.datetime.now(tz=datetime.timezone.utc)
        + datetime.timedelta(minutes=15)},
    "secret", algorithm="HS256",
)
jwt.decode(token, "secret", algorithms=["HS256"])  # raises ExpiredSignatureError once past exp
```

## Architecture / How It Works

PyJWT is organized around a small set of building blocks:

- **`jwt.encode(payload, key, algorithm, headers=...)`** serializes the payload to compact JWS form: `base64url(header).base64url(payload).base64url(signature)`. Since 2.0 it returns a `str`; older 1.x returned `bytes`, a common breakage during upgrades.
- **`jwt.decode(token, key, algorithms=[...], options=..., audience=..., issuer=..., leeway=...)`** splits the token, checks the signature against the allowlist, then validates registered claims.
- **Algorithm registry** (`jwt.algorithms`) — a pluggable table mapping `alg` names to implementations. `HMACAlgorithm` uses the stdlib `hmac`/`hashlib` and always ships. `RSAAlgorithm`, `ECAlgorithm`, `Ed25519Algorithm`, and RSA-PSS require the `cryptography` package; without it, requesting those algorithms raises rather than silently degrading.
- **`PyJWKClient`** — fetches a JWKS (JSON Web Key Set) document from an endpoint (e.g. an OIDC provider's `/.well-known/jwks.json`), selects the signing key by `kid`, and returns it for `decode()`. It performs blocking HTTP and caches keys in memory.

Claim validation covers the RFC 7519 registered claims: `exp` (expiration), `nbf` (not-before), `iat` (issued-at), `aud` (audience, checked only if you pass `audience=`), and `iss` (issuer, checked only if you pass `issuer=`). A `leeway` parameter absorbs clock skew. Each check can be toggled via the `options` dict (`verify_signature`, `verify_exp`, `require`, etc.).

The security model rests on the caller never trusting the token's own header. `decode()` reads the `alg` header only to pick a verifier *from the allowlist you supplied*; an `alg` not in `algorithms=` is rejected before any cryptography runs. The `none` algorithm is only honored when the key is `None` and `"none"` is explicitly allowed.

## Production Notes

**Always pass `algorithms=` explicitly, and keep it to one family.** This is not optional style. Since 2.0 `decode()` raises if `algorithms` is omitted. If you accept both symmetric and asymmetric algorithms in the same allowlist while verifying against a public key, you reopen the RS256/HS256 confusion class of bug (CVE-2022-29217, fixed in 2.4.0 by rejecting asymmetric keys used as HMAC secrets)[^2]. Pick the exact algorithm(s) you actually issue.

**The `cryptography` dependency is a hard gate, not a nicety.** A base `pip install pyjwt` cannot verify RS256/ES256/EdDSA — it raises at decode time. Deployments that work in dev (HS256) and fail in prod (RS256 from an OIDC provider) almost always forgot `pyjwt[crypto]`. Pin the extra in your lockfile.

**JWTs are not encrypted and not revocable.** The payload is readable by anyone; never put secrets in it. Because verification is stateless, you cannot invalidate an issued token before its `exp` without an external denylist. The standard mitigation is short-lived access tokens plus a separate refresh mechanism.

**`PyJWKClient` needs cache discipline.** It does synchronous HTTP and caches keys, but if you construct a fresh client per request you hammer the JWKS endpoint and add latency and a hard external dependency to your auth path. Construct it once at startup and reuse it; be aware key rotation means a `kid` miss triggers a refetch.

**Upgrade pain lives in the 1.x → 2.x jump.** 2.0 (2020) dropped Python 2, made `algorithms` mandatory, and changed `encode()` to return `str` instead of `bytes` — code doing `token.decode("utf-8")` breaks. The old `verify=False` keyword became `options={"verify_signature": False}`. Skipping signature verification still runs claim checks unless you also disable those; read the options semantics carefully before turning anything off.

**Timezone-aware datetimes.** Passing naive `datetime` objects for `exp`/`nbf` has been a recurring source of off-by-timezone bugs; use `datetime.now(tz=timezone.utc)`.

## When to Use / When Not

**Use when:**
- You need to issue or verify signed JWTs (HS/RS/ES/EdDSA) and nothing more.
- You are a client of an OIDC/OAuth provider and need to validate their tokens via JWKS.
- You want a small, auditable, widely-vetted dependency with minimal transitive weight.

**Avoid when:**
- You need encrypted tokens (JWE), key wrapping, or full JWK/JWS/JOSE tooling — PyJWT does none of it.
- You want a complete OAuth2/OIDC framework (client or server) rather than a token primitive.
- You are tempted to build auth from scratch and would be better served by a batteries-included framework's session handling.

## Alternatives

- mpdavis/python-jose — broader JOSE coverage including JWE/JWK/JWS; use when you need encryption or full key-set handling, though it is less actively maintained than PyJWT.
- latchset/jwcrypto — complete JOSE (JWS/JWE/JWK) built on `cryptography`; use when you need encrypted tokens or spec-complete key management.
- authlib/authlib — full OAuth 1/2 and OIDC framework with its own JWT support; use when you are building an auth server/client, not just handling tokens.
- lepture/joserfc — a newer, RFC-focused JOSE implementation; use when you want modern JWE/JWK support with an actively maintained codebase.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2011-02 | Initial implementation by @progrium. |
| 1.0 | 2015 | API stabilization; explicit-algorithm safety work begins. |
| 1.5 | 2017 | Broader algorithm support via `cryptography`. |
| 2.0 | 2020-12 | Dropped Python 2; `algorithms=` required in `decode()`; `encode()` returns `str`[^3]. |
| 2.4.0 | 2022-05 | Fix for key-confusion vulnerability CVE-2022-29217[^2]. |
| 2.8.0 | 2023 | EdDSA and maintenance; expanded claim options. |
| 2.9.0 | 2024 | Python version support refresh. |
| 2.10.x | 2024 | Latest line; includes an issuer-validation hardening fix (CVE-2024-53861)[^4]. |

## References

[^1]: M. Jones, J. Bradley, N. Sakimura, "JSON Web Token (JWT)" — RFC 7519, 2015-05. https://datatracker.ietf.org/doc/html/rfc7519
[^2]: GitHub Security Advisory GHSA-ffqj-6fqr-9h24 / CVE-2022-29217 — "Key confusion through non-blocklisted public key formats", PyJWT. https://github.com/jpadilla/pyjwt/security/advisories/GHSA-ffqj-6fqr-9h24
[^3]: PyJWT changelog, 2.0.0. https://pyjwt.readthedocs.io/en/stable/changelog.html
[^4]: GitHub Security Advisory for CVE-2024-53861 — issuer validation in PyJWT. https://github.com/jpadilla/pyjwt/security/advisories

## Tags

python, jwt, jose, authentication, security, rfc-7519, tokens, oidc, cryptography, library
