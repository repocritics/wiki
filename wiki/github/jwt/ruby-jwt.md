# jwt/ruby-jwt

> The de facto Ruby implementation of RFC 7519 JSON Web Tokens — the gem nearly every Ruby auth library ultimately depends on.

[GitHub repo](https://github.com/jwt/ruby-jwt) ·
[Official website](http://ruby-jwt.org) ·
[License: MIT](https://github.com/jwt/ruby-jwt/blob/main/LICENSE)

## Overview

`ruby-jwt` (the `jwt` gem) encodes and decodes JSON Web Tokens per RFC 7519, using
Ruby's bundled OpenSSL for the actual cryptography[^1]. It is pure token codec: it
signs, verifies, and validates the reserved JWT claims, and does nothing about
sessions, users, or OAuth flows. That narrow scope is why it sits underneath most of
the Ruby authentication ecosystem — Devise-JWT, Doorkeeper's OIDC layer, Knock, and
countless hand-rolled API auth setups delegate the token work here. The repository
dates to 2011 and is still actively released (multiple versions in 2026), which for a
security-sensitive dependency is the property that matters most.

The library's defining tension is between ergonomics and cryptographic safety. JWT as
a format has a long history of implementation vulnerabilities — algorithm-substitution
attacks, the `none` algorithm, and RSA-public-key-as-HMAC-secret confusion[^2]. Much
of `ruby-jwt`'s API design is a reaction to that history: it forces you to name the
expected algorithm at decode time rather than trusting the token's own header. This
makes the safe path slightly more verbose than competing libraries in other languages,
and the docs repeatedly warn against the shortcuts that reintroduce the classic holes.

As of v3.0 (2025) the gem carries two parallel APIs: the original functional
`JWT.encode`/`JWT.decode` pair, and a newer object model (`JWT::Token`,
`JWT::EncodedToken`) that exposes signing, claim verification, and JWK key-finding as
composable steps[^3]. Both are supported; the 2.x line is still maintained in parallel.

## Getting Started

```bash
gem install jwt          # or: gem 'jwt' in your Gemfile
```

```ruby
require 'jwt'

hmac_secret = ENV.fetch('JWT_SECRET')
payload     = { sub: 'user-42', exp: Time.now.to_i + 3600 }

token = JWT.encode(payload, hmac_secret, 'HS256')

begin
  # 3rd arg = verify; ALWAYS pin the algorithm to block substitution attacks
  decoded, header = JWT.decode(token, hmac_secret, true, { algorithm: 'HS256' })
  decoded['sub']  # => "user-42"
rescue JWT::ExpiredSignature
  # token past its exp claim
rescue JWT::DecodeError
  # bad signature, malformed token, wrong algorithm, etc.
end
```

The single most important rule: never pass the algorithm dynamically from the token's
own header. Hard-code the algorithm(s) you accept, or an attacker can downgrade an
RSA-signed token to `none` or to an HMAC keyed on your public key[^2].

## Architecture / How It Works

The core is small. `JWT.encode` builds the `header.payload.signature` triple:
base64url-encode the JSON header and payload, join with `.`, and sign that string with
the algorithm's `sign` method. `JWT.decode` reverses it, then runs two independent
stages — **signature verification** and **claim verification** — which the newer object
API makes explicit as separate calls.

Algorithms are pluggable objects under `JWT::JWA`. Natively supported (all via OpenSSL):
`none`, HMAC (HS256/384/512), RSA (RS256/384/512), ECDSA (ES256/384/512/256K), and
RSASSA-PSS (PS256/384/512). EdDSA was moved out to the separate `jwt-eddsa` gem in v3.0
to keep the core free of the `RbNaCl`/libsodium dependency[^3]. A custom algorithm is
any object that extends `JWT::JWA::SigningAlgorithm` and implements `alg` plus
`sign`/`verify` — the mechanism by which extension gems slot in.

Key handling is the other half. `JWT::JWK` imports and exports keys in JWK format;
`JWT::JWK::Set` models a JWKS document; and `JWT::JWK::KeyFinder` (or any object
responding to `#call`) resolves the right key from a token's `kid` header at verify
time. This is what makes OIDC-style verification against a rotating remote JWKS
endpoint practical without hand-parsing key sets.

Claim validation covers the RFC reserved claims — `exp`, `nbf`, `iss`, `aud`, `jti`,
`iat`, `sub` — each toggled and parameterized through the decode options hash (e.g.
`verify_iss: true, iss: 'https://issuer'`). Issuer and JTI verification accept procs
and regexps, so replay protection and custom issuer rules are expressible inline. Every
failure raises a specific subclass of `JWT::DecodeError` (`JWT::ExpiredSignature`,
`JWT::InvalidIssuerError`, `JWT::InvalidAudError`, …), which is the intended control-flow
surface.

## Production Notes

- **Algorithm pinning is not optional.** The `algorithm:` option on `decode` is the
  primary defense against the well-documented JWT attack class[^2]. Passing an array
  (`algorithm: ['RS256', 'RS512']`) is fine; deriving it from the incoming token is a
  vulnerability. Code review this line specifically.
- **`exp` is verified by default; `nbf`/`iss`/`aud`/`sub`/`jti` are not.** A token with
  a bogus `iss` still decodes cleanly unless you opt into `verify_iss`. Teams routinely
  assume more is checked than actually is — enable each claim you rely on explicitly.
- **Clock skew.** Use `exp_leeway`/`nbf_leeway` (a few minutes max) for distributed
  systems; do not disable expiry checks to paper over skew.
- **Two maintained major lines.** 3.x (new object API, Ruby 2.5+ era baseline, EdDSA
  extracted) and 2.x (2.10.x still shipping in 2026) coexist[^4]. The v2→v3 upgrade
  guide flags behavioral changes; read it rather than bumping blindly, especially if you
  used EdDSA (now requires the `jwt-eddsa` gem) or relied on internal `JWT::JWA` layout.
- **JWT is signed, not encrypted.** Payloads are base64url, i.e. plaintext to anyone
  holding the token. Never put secrets in a JWT. This gem does not implement JWE
  (encryption) at all — reach for `json-jwt` or `ruby-jose` if you need that.
- **`none` algorithm exists and is dangerous.** It is supported for spec completeness
  and detached-payload use cases, but a decode path that ever accepts `'none'` accepts
  unsigned tokens. Keep it out of your allowed-algorithm list in production.
- **OpenSSL is the whole crypto backend.** Behavior and available curves track the
  OpenSSL your Ruby is linked against; ES256K and some edge cases vary across OpenSSL 1.1
  vs 3.x builds.

## When to Use / When Not

**Use when:**
- You need to issue or verify signed JWTs in Ruby and want the standard, audited,
  broadly-depended-on implementation.
- You're verifying third-party OIDC/OAuth tokens against a JWKS endpoint (`kid` + key
  finder support is first-class).
- You want claim validation (`exp`, `aud`, `iss`, replay via `jti`) without writing it.

**Avoid when:**
- You need encrypted tokens (JWE) or a full JOSE toolkit — this gem only does JWS/JWT.
- You want a turnkey Rails auth system — this is a codec, not a framework; pair it with
  Devise/Doorkeeper or build the session layer yourself.
- You only ever verify one vendor's tokens (e.g. Google) — a purpose-built verifier may
  be simpler than wiring up JWKS handling.

## Alternatives

- nov/json-jwt — use instead when you need JWE encryption or a fuller JOSE surface, not just signed tokens.
- potatosalad/ruby-jose — use when you want a complete JOSE toolkit (JWK/JWE/JWS) with algorithm agility.
- doorkeeper-gem/doorkeeper — use when you need a full OAuth 2.0 / OIDC provider, not a raw token codec.
- waiting-for-code/devise-jwt — use when bolting JWT sessions onto a Devise/Rails app.
- google/google-auth-library-ruby — use when you only verify Google-issued ID tokens.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2011-02-23 | Project origin on GitHub[^1]. |
| 2.0.x | 2017 (approx) | Rewrite of the decode/verify path; stricter algorithm handling. |
| 2.7.0 | 2023-02-01 | Continued 2.x maintenance line[^4]. |
| 2.9.0 | 2024-09-15 | Late-2.x feature/security releases[^4]. |
| 3.0.0.beta1 | 2025-01-25 | Preview of the object API (`Token`/`EncodedToken`). |
| 3.0.0 | 2025-06-14 | New object model; EdDSA split to `jwt-eddsa` gem[^3]. |
| 2.10.3 | 2026-05-22 | 2.x line still maintained alongside 3.x[^4]. |
| 3.2.0 | 2026-05-13 | Current 3.x release[^4]. |

## References

[^1]: `jwt/ruby-jwt` repository, "A ruby implementation of the RFC 7519 OAuth JSON Web Token (JWT) standard." https://github.com/jwt/ruby-jwt — RFC 7519: https://tools.ietf.org/html/rfc7519
[^2]: Auth0, "Critical vulnerabilities in JSON Web Token libraries" (alg confusion, `none`, RSA/HMAC key confusion). https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/
[^3]: ruby-jwt README — `JWT::Token`/`JWT::EncodedToken` object API, custom algorithms, and EdDSA via the `jwt-eddsa` gem. https://github.com/jwt/ruby-jwt#readme
[^4]: ruby-jwt release tags (2.x and 3.x lines shipping in parallel). https://github.com/jwt/ruby-jwt/releases — upgrade guide: https://github.com/jwt/ruby-jwt/blob/main/UPGRADING.md

## Tags

ruby, jwt, json-web-token, authentication, authorization, oauth, jwk, cryptography, rfc-7519, security, token
