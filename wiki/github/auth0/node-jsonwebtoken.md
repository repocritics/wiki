# auth0/node-jsonwebtoken

> The de facto Node.js implementation of JSON Web Tokens (RFC 7519) — the `jsonwebtoken` npm package most stateless-auth backends reach for first.

[GitHub repo](https://github.com/auth0/node-jsonwebtoken) ·
[npm: jsonwebtoken](https://www.npmjs.com/package/jsonwebtoken) ·
[License: MIT](https://github.com/auth0/node-jsonwebtoken/blob/master/LICENSE)

## Overview

`jsonwebtoken` is a small library that signs, verifies, and decodes JWTs. It
was first published in 2013 and is maintained by Auth0 (now part of Okta)[^1].
Despite being one of the oldest options, it remains the default in most Node
tutorials and boilerplates, which is why it is downloaded tens of millions of
times per week and depended on by a very large slice of the npm ecosystem.
Development has slowed to maintenance pace — the last significant release was
the 9.x line — but the code it wraps is mature and the API surface is small.

The library is deliberately thin: it is a convenience layer over
`brianloveswords/node-jws` (and `jwa` below that) for the actual signing and
HMAC/RSA/ECDSA crypto, plus claim helpers (`exp`, `nbf`, `aud`, `iss`, `sub`)
and a set of typed error classes[^2]. It does not do JWKS fetching, key
rotation, JWE (encrypted tokens), or general JOSE — it is JWS/JWT only.

The defining tension of this library is that its historically permissive
defaults have been the source of real JWT vulnerabilities across the whole
ecosystem, and the 9.0 release (a hard security cleanup) is still not adopted
everywhere. A great deal of production code is pinned to 8.5.1 and inherits the
older, looser behavior. Reading the version number matters more here than for
almost any other dependency.

## Getting Started

```bash
npm install jsonwebtoken
```

```js
const jwt = require("jsonwebtoken");

// Sign — HMAC SHA-256 by default (HS256)
const token = jwt.sign({ sub: "user-123", role: "admin" }, process.env.JWT_SECRET, {
  expiresIn: "1h",          // "vercel/ms" span or seconds count
  issuer: "api.example.com",
});

// Verify — ALWAYS pin the algorithm list on the verify side
try {
  const payload = jwt.verify(token, process.env.JWT_SECRET, {
    algorithms: ["HS256"],
    issuer: "api.example.com",
  });
  console.log(payload.sub); // "user-123"
} catch (err) {
  // TokenExpiredError | NotBeforeError | JsonWebTokenError
}
```

For RSA/ECDSA, pass a PEM private key to `sign` with an explicit `algorithm`
(e.g. `RS256`), and the corresponding public key/cert to `verify`.

## Architecture / How It Works

The public API is three functions. `sign` builds the header, applies claim
options onto the payload, and hands the assembled parts to `jws.sign`.
`verify` splits the compact token on `.`, decodes header and payload, checks
that the token's `alg` is in the caller's allowed list, verifies the signature
via `jws.verify`, then runs claim validation (`exp`, `nbf`, `aud`, `iss`,
`sub`, `jti`, `maxAge`) with `clockTolerance` and `clockTimestamp` knobs.
`decode` does the same parsing with no signature check at all — a footgun if
mistaken for `verify`[^2].

By default `sign` and `verify` are synchronous and return/throw directly.
Passing a callback makes them asynchronous; in that mode `verify`'s
`secretOrPublicKey` may be a function `(header, callback)` that resolves a key
by `header.kid` — the intended integration point for `auth0/node-jwks-rsa`
against an OIDC provider's JWKS endpoint. There is no built-in key cache or
JWKS client; that is a separate dependency by design.

Because the actual crypto lives in `jws`/`jwa`, this repo is mostly input
validation and claim bookkeeping. That is also where its security history is
concentrated: the choice of which algorithm to trust at verification time is a
policy decision, and for years the library made a permissive default choice.

## Production Notes

**Always pass `algorithms` to `verify`.** The classic JWT attacks are algorithm
confusion (an attacker changes `alg` from `RS256` to `HS256` and signs with the
public key as the HMAC secret) and the `alg: none` bypass[^3]. If you omit the
`algorithms` option, older versions infer allowed algorithms from the key type,
which historically permitted these class of attacks. Pin an explicit allowlist
on every `verify` call and never accept `none`.

**Version matters — 8.x vs 9.x is a security boundary.** The 9.0.0 release
(late 2022) landed a batch of breaking hardening changes: it rejects RSA keys
with a modulus under 2048 bits unless `allowInsecureKeySizes` is set, tightens
`secretOrPublicKey` handling, validates that key type matches the declared
algorithm, and drops support for very old Node versions[^4]. These correspond
to CVEs disclosed in December 2022 (GHSA advisories covering forgeable tokens
and insecure key handling). Codebases still on 8.5.1 do not have these
protections; migrating requires reading the v8→v9 migration notes because the
stricter checks will reject keys and tokens that previously "worked."

**`decode` is not `verify`.** `jwt.decode()` returns the payload without
checking the signature. It is only safe for reading a token you already trust
(e.g. to inspect `kid` before verifying). Treat any decoded payload from an
untrusted source as raw user input.

**`expiresIn`/`notBefore` string parsing is a trap.** A bare numeric string
like `"120"` is interpreted as milliseconds (`120ms`), while a plain number
`120` is seconds. Always include units (`"2h"`, `"7d"`) or use integer seconds
to avoid tokens that expire far sooner than intended.

**Clock skew.** `exp`/`nbf` checks are exact to the second; distributed systems
with unsynchronized clocks should set `clockTolerance` (a few seconds) to avoid
spurious `NotBeforeError`/`TokenExpiredError` at the boundaries.

**No refresh-token machinery.** The library intentionally ships nothing for
refresh/rotation; the maintainers point at a community gist rather than
building it in. Session lifecycle is your problem.

## When to Use / When Not

**Use when:**
- You need straightforward HS/RS/ES signing and verification in Node and want
  the most widely understood, best-documented option.
- You are verifying third-party OIDC tokens and will pair it with `jwks-rsa`.
- You want a callback/synchronous CommonJS API and don't need ESM/Web Crypto.

**Avoid when:**
- You need JWE (encrypted tokens), full JOSE, or key-management primitives —
  reach for `jose` instead.
- You want a modern ESM-first, Web-Crypto-based library that also runs on
  Deno, Bun, Cloudflare Workers, and browsers — `jose` targets those runtimes;
  `jsonwebtoken` is Node-only.
- Raw verify throughput is critical on a hot path — `fast-jwt` benchmarks
  faster.

## Alternatives

- panva/jose — modern JOSE toolkit (JWT, JWS, JWE, JWK), ESM, Web Crypto,
  cross-runtime. Use instead when you want encrypted tokens, JWKS handling, or
  non-Node runtimes.
- nearform/fast-jwt — performance-focused JWT for Node. Use when verification
  throughput dominates and you can accept a smaller ecosystem.
- auth0/express-jwt — Express middleware wrapper around this library. Use when
  you just want request-level auth middleware, not the low-level primitives.
- cisco/node-jose — full JOSE implementation. Use when you need JWE/JWK
  management and prefer it over `jose`.
- kjur/jsrsasign — pure-JS crypto covering JWT plus X.509/PKCS. Use when you
  need broad crypto in environments without native bindings.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2013-07 | First publish; wrapper over `node-jws`[^1]. |
| 5.0.0 | 2015 | Algorithm-confusion / `alg:none` class of issues addressed industry-wide[^3]. |
| 8.0.0 | 2017 | API cleanup; `8.5.1` became a long-lived pinned baseline. |
| 9.0.0 | 2022-12 | Security hardening: key-size checks, stricter algorithm/key validation, dropped old Node; breaking[^4]. |

## References

[^1]: `jsonwebtoken` on npm — package metadata and maintainer (Auth0). https://www.npmjs.com/package/jsonwebtoken
[^2]: Project README — `sign`/`verify`/`decode` API, options, and error classes (`TokenExpiredError`, `JsonWebTokenError`, `NotBeforeError`). https://github.com/auth0/node-jsonwebtoken#readme
[^3]: Tim McLean, "Critical vulnerabilities in JSON Web Token libraries" (2015) — `alg:none` and RS256/HS256 confusion. https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/
[^4]: Migration Notes: v8 to v9 (key-size and algorithm-validation breaking changes). https://github.com/auth0/node-jsonwebtoken/wiki/Migration-Notes:-v8-to-v9

## Tags

javascript, nodejs, jwt, authentication, authorization, security, jose, jws, tokens, oauth, npm-package
