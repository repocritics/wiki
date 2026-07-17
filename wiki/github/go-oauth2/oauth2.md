# go-oauth2/oauth2

> An OAuth 2.0 authorization-server (provider) library for Go — you host the token endpoint, you do not consume one.

[GitHub repo](https://github.com/go-oauth2/oauth2) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/github.com/go-oauth2/oauth2/v4) ·
[License: MIT](https://github.com/go-oauth2/oauth2/blob/master/LICENSE)

## Overview

go-oauth2 is a server-side implementation of the OAuth 2.0 authorization framework (RFC 6749[^1]) for Go, first published in 2016 by the author known as "Lyric"[^2]. It gives you the pieces to run your own authorization server: an authorize endpoint, a token endpoint, pluggable client/token storage, and access-token generation. It has ~3,600 stars and ~590 forks, making it the most-starred pure-Go OAuth *provider* library, though it is considerably smaller than the client-side ecosystem it is often confused with.

The single most important thing to know: **this is not `golang.org/x/oauth2`.** The `x/oauth2` package is the official *client* library for talking to Google, GitHub, and other providers. go-oauth2/oauth2 is the opposite side of the wire — you use it when *you* are the provider issuing tokens to third-party clients. The naming collision causes a large fraction of the confusion in issues and Stack Overflow answers.

The defining tradeoff is scope. go-oauth2 implements the OAuth 2.0 grant flows and little else: it deliberately does not ship OpenID Connect (no `id_token`, no discovery document, no userinfo endpoint), no login UI, no consent screen, and no client-registration admin. That minimalism makes it the fastest way to stand up a basic token endpoint, and the wrong choice if you actually need a spec-strict, OIDC-capable identity provider.

## Getting Started

```bash
go get -u github.com/go-oauth2/oauth2/v4
```

```go
package main

import (
	"log"
	"net/http"

	"github.com/go-oauth2/oauth2/v4/manage"
	"github.com/go-oauth2/oauth2/v4/models"
	"github.com/go-oauth2/oauth2/v4/server"
	"github.com/go-oauth2/oauth2/v4/store"
)

func main() {
	manager := manage.NewDefaultManager()
	manager.MustTokenStorage(store.NewMemoryTokenStore()) // in-memory: dev only

	clientStore := store.NewClientStore()
	clientStore.Set("000000", &models.Client{
		ID:     "000000",
		Secret: "999999",
		Domain: "http://localhost",
	})
	manager.MapClientStorage(clientStore)

	srv := server.NewDefaultServer(manager)
	srv.SetClientInfoHandler(server.ClientFormHandler)
	srv.UserAuthorizationHandler = func(w http.ResponseWriter, r *http.Request) (string, error) {
		return "000000", nil // replace with your real login/consent flow
	}

	http.HandleFunc("/authorize", func(w http.ResponseWriter, r *http.Request) {
		if err := srv.HandleAuthorizeRequest(w, r); err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
		}
	})
	http.HandleFunc("/token", func(w http.ResponseWriter, r *http.Request) {
		srv.HandleTokenRequest(w, r)
	})

	log.Fatal(http.ListenAndServe(":9096", nil))
}
```

The import path is versioned: the current major is `/v4`. Code, examples, or answers referencing bare `github.com/go-oauth2/oauth2` (no `/v4`) target a pre-modules version and will not compile against current releases.

## Architecture / How It Works

The library is organized as a set of interfaces wired together by a `Manager`:

- **`server.Server`** — parses HTTP requests, drives the grant flows (authorization code, client credentials, password, implicit, refresh token), and writes RFC-shaped JSON responses. You register `HandleAuthorizeRequest` and `HandleTokenRequest` on your own mux; it is transport-agnostic beyond `net/http`.
- **`manage.Manager`** — the orchestration core. It owns the token-generation strategy, TTLs, and the binding between clients, tokens, and stores. `NewDefaultManager()` sets conventional defaults you then override.
- **`store`** — two separate concerns: a `ClientStore` (registered clients + secrets) and a `TokenStore` (issued access/refresh tokens). The bundled stores are `NewMemoryTokenStore()` and a BuntDB-backed file store; everything else (Redis, MongoDB, MySQL, Postgres, DynamoDB, GORM/XORM, Firestore) lives in separate community repos under and around the `go-oauth2` org[^3].
- **`generates`** — how tokens are minted. The default produces opaque random strings that must be looked up in the `TokenStore` on every resource request. `NewJWTAccessGenerate` instead issues self-contained signed JWTs, trading revocability for statelessness.

The pivotal design decision is the **opaque-token-by-default** model: an access token is just a key into the store, so validation means a store round-trip, but revocation is immediate (delete the row). Switching to JWT access tokens inverts both properties — no store lookup, but a leaked or over-privileged token stays valid until it expires because nothing consults a database to reject it.

What the library intentionally leaves to you: authenticating the end user (`UserAuthorizationHandler` is a blank hook), rendering login and consent screens, registering and managing clients, hashing client secrets, and any scope-authorization policy beyond string matching.

## Production Notes

The quick-start is a demonstration, not a template. Several defaults are actively unsafe for production:

- **In-memory token store is not durable and not shared.** `NewMemoryTokenStore()` loses every token on restart and cannot be used across more than one instance. Any real deployment needs Redis, a SQL store, or Mongo. This also means horizontal scaling is a storage decision, not a code decision.
- **`SetAllowGetAccessRequest(true)`** (shown in the upstream README) permits token requests over `GET`, which puts `client_secret` in the URL query string — where it lands in access logs, proxies, and browser history. Leave it off; require `POST`.
- **Client secrets are stored and compared in plaintext.** The bundled `ClientStore` holds `Secret` as a raw string with no hashing. If you persist clients, you own hashing and constant-time comparison.
- **The README's JWT example imports `github.com/dgrijalva/jwt-go`, which is deprecated and unmaintained** (it carries CVE-2020-26160 for missing `aud` validation). The maintained successor is `github.com/golang-jwt/jwt`. Audit your `go.mod` and the `generates` JWT path so you are not pulling the abandoned fork transitively.
- **No OIDC.** There is no `id_token`, discovery endpoint, or userinfo endpoint. Bolting OIDC on top of go-oauth2 by hand is a common source of subtly non-conformant providers; if clients expect OpenID Connect, pick a library that ships it.
- **Implicit grant is present but obsolete.** OAuth 2.0 Security BCP and OAuth 2.1 both discourage or remove it; prefer authorization code with PKCE. Confirm PKCE (`code_challenge`) handling on the version you pin before relying on it — support has evolved over releases.

On maintenance: the project is real but slow-moving. The default branch was last pushed in 2025-08 and there are ~116 open issues against a small maintainer group. It is stable rather than actively developed — fine for a mature RFC 6749 surface, but do not expect rapid response to new spec drafts or CVEs in the dependency tree.

## When to Use / When Not

**Use when:**
- You need to *issue* OAuth 2.0 tokens (you are the authorization server), not consume them.
- You want a minimal, embeddable token endpoint inside an existing Go service with your own login flow.
- Plain OAuth 2.0 grants are enough and you do not need OpenID Connect.

**Avoid when:**
- You actually want an OAuth *client* — use `golang.org/x/oauth2` instead.
- You need OIDC (`id_token`, discovery, userinfo), dynamic client registration, or a hardened, spec-audited server.
- You want a deployable, batteries-included identity provider rather than a library to assemble one.

## Alternatives

- ory/fosite — lower-level, security-first OAuth2 + OIDC framework; more wiring, but spec-strict and audited. Use when correctness and OIDC matter more than time-to-first-token.
- ory/hydra — a deployable headless OAuth2/OIDC server (built on fosite). Use when you want to run a server, not embed a library.
- zitadel/oidc — OIDC-focused Go library. Use when you need OpenID Connect specifically.
- dexidp/dex — OIDC provider with upstream connectors (LDAP, SAML, GitHub). Use as a federation broker in front of existing identity sources.
- golang.org/x/oauth2 — the official *client* library. Use when you are integrating with someone else's provider, not building your own.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-05-26 | First commit; OAuth 2.0 server library by "Lyric"[^2]. |
| v3.x | ~2018 | Pre-Go-modules line, imported without a version suffix. |
| v4.x | Go-modules era | Current major; import path `github.com/go-oauth2/oauth2/v4`. |
| — | 2025-08-20 | Latest push to `master` at time of writing[^4]. |

## References

[^1]: D. Hardt, ed., "The OAuth 2.0 Authorization Framework," RFC 6749, IETF, 2012. https://datatracker.ietf.org/doc/html/rfc6749
[^2]: go-oauth2/oauth2 README and LICENSE — "Copyright (c) 2016 Lyric." https://github.com/go-oauth2/oauth2
[^3]: go-oauth2/oauth2 README, "Store Implements" — external store backends (Redis, MongoDB, MySQL, PostgreSQL, DynamoDB, XORM, GORM, Firestore, Hazelcast). https://github.com/go-oauth2/oauth2#store-implements
[^4]: GitHub repository metadata, `pushed_at` 2025-08-20. https://github.com/go-oauth2/oauth2

## Tags

go, oauth2, authorization-server, oauth2-provider, authentication, security, rfc6749, jwt, backend, library
