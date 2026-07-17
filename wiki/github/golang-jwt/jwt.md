# golang-jwt/jwt

> The de facto Go implementation of JSON Web Tokens, resurrected by a community team after the original `dgrijalva/jwt-go` was abandoned.

[GitHub repo](https://github.com/golang-jwt/jwt) ·
[Official website](https://golang-jwt.github.io/jwt/) ·
[License: MIT](https://github.com/golang-jwt/jwt/blob/main/LICENSE)

## Overview

golang-jwt/jwt parses, verifies, generates, and signs JSON Web Tokens (RFC 7519) in Go. It is the near-universal choice for JWT handling in the Go ecosystem — most auth middleware, OAuth2 servers, and API gateways written in Go either depend on it directly or expose it as the token backend.

The project exists because of a maintenance transfer. The original library, `dgrijalva/jwt-go`, was the standard for years but went effectively unmaintained around 2021; a dedicated team cloned it into the `golang-jwt` org after the original author signalled he was stepping back[^1]. This history matters operationally: import paths, module versions, and a well-known CVE all trace back to that split, and codebases still carry `dgrijalva/jwt-go` imports that should be migrated.

The library's defining tension is that JWT is a footgun-rich format and this library is a thin, unopinionated wrapper over it. It gives you signing methods and a parser, but the security-critical decisions — which algorithms to accept, how to resolve keys, whether to trust the `alg` header — are left to the caller. The API has been progressively hardened (v5 made validation stricter by default) but it still assumes you understand the algorithm-confusion class of attacks[^2].

## Getting Started

```sh
go get -u github.com/golang-jwt/jwt/v5
```

```go
package main

import (
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

func main() {
	secret := []byte("replace-with-a-real-secret")

	// Sign
	claims := jwt.MapClaims{
		"sub": "1234567890",
		"exp": time.Now().Add(time.Hour).Unix(),
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	signed, _ := token.SignedString(secret)

	// Verify — the Keyfunc MUST constrain the algorithm family.
	parsed, err := jwt.Parse(signed, func(t *jwt.Token) (interface{}, error) {
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected alg: %v", t.Header["alg"])
		}
		return secret, nil
	})
	if err != nil {
		panic(err)
	}
	fmt.Println(parsed.Valid, parsed.Claims.(jwt.MapClaims)["sub"])
}
```

The `SigningMethodHMAC` type assertion in the Keyfunc is not optional boilerplate — omitting it is the root of the classic RS256→HS256 key-confusion attack, where an attacker signs a token with the public RSA key as an HMAC secret.

## Architecture / How It Works

The core abstraction is the `SigningMethod` interface (`Sign`, `Verify`, `Alg`). Concrete implementations ship for HMAC-SHA (HS256/384/512), RSA (RS256/…), RSA-PSS (PS256/…), ECDSA (ES256/…), and EdDSA (Ed25519). Methods are registered in a global registry via `RegisterSigningMethod`; you can add your own — for HSMs or cloud KMS backends — by implementing the interface, which is how the third-party GCP/AWS/TPM extensions work[^3].

Parsing flows through a `Parser` that: base64url-decodes the three segments, unmarshals header and claims, looks up the `SigningMethod` named by the `alg` header, invokes your `Keyfunc` to obtain the verification key, and then runs signature verification followed by claims validation. The `Keyfunc` is the security seam — it receives the parsed (but not-yet-verified) token so you can inspect `t.Method` before returning a key. Trusting `t.Header["alg"]` blindly here is the primary way applications get compromised.

Claims are an interface, not a struct. `MapClaims` (a `map[string]interface{}`) is the dynamic option; `RegisteredClaims` provides typed standard fields (`iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`) with proper `NumericDate` handling. In v5, custom claim types can implement the `ClaimsValidator` interface to run application-specific checks during parsing. Parser behavior is configured with functional options — `WithValidMethods`, `WithAudience`, `WithIssuer`, `WithExpirationRequired`, `WithLeeway` — which is the idiomatic way to lock down what a token must satisfy.

## Production Notes

**Always pin accepted algorithms.** Use `jwt.WithValidMethods([]string{"HS256"})` (or your allowed set) on the parser in addition to the type check in the Keyfunc. Belt and suspenders here prevents both alg-confusion and `alg=none` acceptance. `alg=none` tokens are rejected unless you explicitly pass the sentinel `jwt.UnsafeAllowNoneSignatureType` as the key — which almost nothing should.

**CVE-2025-30204** — versions before v5.2.2 (and the v4.5.x line before its corresponding patch) allocated memory proportional to the number of period-delimited segments while splitting the token, allowing a crafted header to drive excessive memory use[^4]. If you are on any pre-5.2.2 v5 or an older v4, upgrade. This is separate from the historically famous **CVE-2020-26160**, an audience-claim bypass in the original `dgrijalva/jwt-go` — one more reason to leave that import behind.

**The v3→v4→v5 import path trap.** Because of Go module semantics, the major version is part of the import path: `github.com/golang-jwt/jwt` (v3), `.../jwt/v4`, `.../jwt/v5`. It is entirely possible — and common in large codebases — to have two major versions linked simultaneously without a compile error, doubling the JWT surface area. Audit with `go mod graph | grep jwt`.

**v5 validation is stricter and not fully backward compatible.** `StandardClaims` was replaced by `RegisteredClaims`; the old struct-embedded `Valid()` pattern gave way to the `ClaimsValidator` interface and parser options. Expiration is validated when present but not required unless you opt in with `WithExpirationRequired` — a token with no `exp` will pass by default, which surprises people. Migrations from v4 usually require touching claims types and re-checking validation assumptions[^5].

**Performance is rarely the bottleneck** — RSA verification dominates and is a property of the crypto, not this library. HMAC verification is cheap; ECDSA/EdDSA sit in between. If you verify thousands of RS256 tokens per second, cache parsed public keys and consider ES256/EdDSA for new systems.

## When to Use / When Not

**Use when:**
- You need standard JWT sign/verify in Go and want the library everything else already integrates with.
- Your needs are JWS-style signed tokens (the overwhelmingly common case).
- You want to plug in a custom signer (KMS/HSM) via the `SigningMethod` interface.

**Avoid when:**
- You need JWE (encrypted tokens), JWK set parsing, or broader JOSE support out of the box — this library is JWT/JWS-focused and pushes JWKS to a third-party extension.
- You want a library that makes the safe path the only path with zero configuration — the security burden here still sits with the caller.
- You are still on `dgrijalva/jwt-go` — that is a migration target, not a choice.

## Alternatives

- lestrrat-go/jwx — full JOSE stack (JWT/JWS/JWE/JWK/JWA); use when you need JWE, native JWK set handling, or standards breadth beyond signed tokens.
- go-jose/go-jose — mature JOSE implementation (successor to square/go-jose); use when you want JWE and a spec-complete JOSE layer, as projects like Dex and CoreOS tooling do.
- cristalhq/jwt — zero-dependency, performance-oriented; use when you want a smaller, faster surface and don't need the extension ecosystem.
- gbrlsnchs/jwt — lighter API with typed algorithms; use when you prefer a more constrained interface over the global registry model.
- MicahParks/keyfunc — not a replacement but a companion; use alongside golang-jwt when you need JWKS-backed key resolution.

## History

| Version | Date | Notes |
|---------|------|-------|
| v3.2.0 | 2020 | Last significant release under the original `dgrijalva/jwt-go` before the handoff. |
| fork | 2021-05 | `golang-jwt` org clones jwt-go after dgrijalva/jwt-go#462[^1]. |
| v4.0.0 | 2021-08 | Adds Go module support; retains backward compatibility with v3 tags. |
| v5.0.0 | 2023-03 | Validation overhaul; `RegisteredClaims` replaces `StandardClaims`; functional parser options; not fully backward compatible[^5]. |
| v5.2.2 | 2025-03 | Fixes CVE-2025-30204 (excessive memory allocation during header parsing)[^4]. |

## References

[^1]: "A dedicated team of open source maintainers decided to clone the existing library into this repository." golang-jwt/jwt README; discussion at https://github.com/dgrijalva/jwt-go/issues/462
[^2]: Auth0, "Critical vulnerabilities in JSON Web Token libraries." https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/
[^3]: golang-jwt/jwt README, "Extensions" — `SigningMethod` interface and third-party GCP/AWS/JWKS/TPM integrations. https://github.com/golang-jwt/jwt#extensions
[^4]: CVE-2025-30204 / GHSA-mh63-6h87-95cp — golang-jwt excessive memory allocation during header parsing, fixed in v5.2.2. https://github.com/golang-jwt/jwt/security/advisories/GHSA-mh63-6h87-95cp
[^5]: golang-jwt/jwt MIGRATION_GUIDE.md — upgrading across major versions. https://github.com/golang-jwt/jwt/blob/main/MIGRATION_GUIDE.md

## Tags

go, golang, jwt, authentication, jose, security, tokens, oauth2, cryptography, rfc7519
