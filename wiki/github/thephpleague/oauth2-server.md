# thephpleague/oauth2-server

> A framework-agnostic PHP OAuth 2.0 authorization server that issues signed JWT access tokens and leaves storage to you.

[GitHub repo](https://github.com/thephpleague/oauth2-server) ·
[Official website](https://oauth2.thephpleague.com) ·
[License: MIT](https://github.com/thephpleague/oauth2-server/blob/master/LICENSE)

## Overview

`league/oauth2-server` implements the *server* side of OAuth 2.0: the piece that authenticates clients, runs grant flows, and issues access and refresh tokens. It does not implement OAuth *client* behaviour (for that the League ships the separate `league/oauth2-client`). It was created by Alex Bilbie in 2012 out of the Linkey project at the University of Lincoln, and has been principally maintained by Andy Millington since around 2017[^1]. It is one of the most depended-on OAuth server libraries in the PHP ecosystem — Laravel Passport is a thin wrapper over it, and Symfony, Mezzio, Drupal (`simple_oauth`) and CakePHP integrations all build on the same core[^2].

The defining design decision is that the library is *storage-agnostic and framework-agnostic*. It defines a set of repository interfaces (`ClientRepositoryInterface`, `AccessTokenRepositoryInterface`, `ScopeRepositoryInterface`, and so on) plus entity interfaces, and you implement them against whatever database and HTTP stack you use. All HTTP crosses the API as PSR-7 messages[^3]. This is what makes it reusable, and also what makes it non-trivial to adopt: there is no batteries-included setup, and a working server requires you to write and wire roughly seven repositories plus a key pair before anything runs.

The second defining decision is that access tokens are **JWTs signed with an asymmetric key pair** rather than opaque random strings. This makes token validation stateless at the resource server — verify the signature and expiry, no database round-trip required — at the cost of tokens you cannot silently rotate and a hard dependency on stable key management.

## Getting Started

```bash
composer require league/oauth2-server
```

Generate an RSA key pair and an encryption key first:

```bash
openssl genrsa -out private.key 2048
openssl rsa -in private.key -pubout -out public.key
php -r 'echo base64_encode(random_bytes(32)), PHP_EOL;'  # encryption key
```

```php
use League\OAuth2\Server\AuthorizationServer;
use League\OAuth2\Server\Grant\ClientCredentialsGrant;
use League\OAuth2\Server\CryptKey;
use DateInterval;

$server = new AuthorizationServer(
    $clientRepository,      // your ClientRepositoryInterface impl
    $accessTokenRepository, // your AccessTokenRepositoryInterface impl
    $scopeRepository,       // your ScopeRepositoryInterface impl
    new CryptKey('file://private.key'),
    $encryptionKey          // base64 string or defuse Key
);

$server->enableGrantType(new ClientCredentialsGrant(), new DateInterval('PT1H'));

// In your /access_token controller (PSR-7 request/response in, response out):
return $server->respondToAccessTokenRequest($request, $response);
```

Protecting an API is a separate `ResourceServer` plus its `AuthorizationValidator` PSR-7 middleware, configured with the public key.

## Architecture / How It Works

The library splits cleanly into two objects. `AuthorizationServer` handles the token *issuance* endpoints; `ResourceServer` handles token *validation* on protected routes. They share only the key material and the `AccessTokenRepository`.

Grants are pluggable strategy objects. You call `enableGrantType()` for each one you want; the server dispatches an incoming request to whichever grant matches its `grant_type`. Out of the box the shipped grants are: authorization code (with mandatory PKCE for public clients per RFC 7636), client credentials, device authorization (RFC 8628), implicit, refresh token, and resource owner password credentials[^2].

Token cryptography is the load-bearing part. Access tokens are JWTs signed with your private key (via `lcobucci/jwt`); the resource server verifies them with the public key. Because validation is signature-based, `ResourceServer` does not need your token store to *validate* a token — but it does call `AccessTokenRepositoryInterface::isAccessTokenRevoked()` during validation, so revocation is honoured if (and only if) your repository actually records revocations. Authorization codes and refresh tokens are not JWTs; they are JSON payloads encrypted with the separate encryption key, which is why that key matters as much as the signing key.

Everything downstream of the interfaces is yours. The library never touches a database, never renders a login or consent screen, and never manages user sessions — the authorization-code grant hands control back to you via a `UserEntityInterface` lookup and an approval step you implement. This is a correctness feature (the security-sensitive protocol logic is centralised and audited) and an onboarding tax (much of a real deployment is code you write).

## Production Notes

**Key stability is the number-one operational footgun.** The encryption key must be identical across every instance and must not change while any refresh token or auth code is outstanding — rotating it invalidates all of them instantly. Rotating the *signing* key invalidates all live access tokens. Neither key has a built-in rotation-with-overlap mechanism, so zero-downtime rotation is a manual exercise (accept two keys during a window). Store both keys outside the repo and outside the token store.

**JWT access tokens cannot be short-circuited to "logged out".** Because a resource server can validate a token offline, the only revocation path is the `isAccessTokenRevoked()` check — which reintroduces the database round-trip that stateless tokens were meant to avoid. Teams that want instant revocation and therefore query the store on every request lose the main benefit of the JWT design; teams that skip it must live with tokens valid until expiry. Keep access-token TTLs short (minutes, not days) and lean on refresh tokens.

**Implicit and password grants are deprecated by the protocol, not just by fashion.** The OAuth 2.0 Security BCP and the OAuth 2.1 draft remove both the implicit grant and the resource owner password credentials grant[^4]. The library still ships them for backward compatibility, but new deployments should use authorization code + PKCE instead. Enabling the password grant in particular means your OAuth server is handling raw user passwords, which is the thing OAuth exists to avoid.

**Upgrades track PHP and crypto, and they bite.** Version 8.0 (2019) dropped old PHP versions and moved token handling onto `lcobucci/jwt`; the 9.0 line (2024) dropped PHP < 8.1 and again bumped the JWT dependency, whose own major versions changed the signer/key API in ways that surface through this library's configuration. Check the `UPGRADE` and changelog notes before a major bump — the public surface is stable, but the transitive JWT dependency is where breakage hides. The current 9.x line requires PHP 8.2–8.5 plus the `openssl` and `json` extensions.

**It is actively maintained and audited.** 9.4.1 shipped in June 2026, and the codebase received a security audit funded by the Mozilla Secure Open Source Fund[^1]. With roughly 6,700 stars and used transitively by Laravel Passport, its real install base is far larger than the star count implies.

## When to Use / When Not

**Use when:**
- You are building an OAuth 2.0 *provider* (issuing tokens to third-party or first-party clients) and want spec-compliant, audited grant logic.
- You are on a PSR-7 stack (Slim, Mezzio, Laminas) or a framework without a bundled OAuth server and want to wire storage yourself.
- You want stateless resource-server validation and are comfortable owning key management.

**Avoid when:**
- You only need to *consume* someone else's OAuth (log in with Google/GitHub) — use an OAuth client library, not this.
- You want a turnkey identity provider with login UI, user management, MFA, and admin console — run a dedicated IdP (Keycloak, Ory Hydra, Authentik) instead.
- You need OpenID Connect out of the box — this is OAuth 2.0 only; OIDC requires an add-on layer such as `steverhoades/oauth2-openid-connect-server`.
- You are on Laravel — use Laravel Passport, which wraps this library with the migrations, routes, and models already written.

## Alternatives

- laravel/passport — use instead when you are on Laravel and want this library's engine with the boilerplate (storage, routes, UI) already provided.
- ory/hydra — use instead when you want a standalone, language-agnostic OAuth2 + OIDC server process rather than an in-app library.
- keycloak/keycloak — use instead when you need a full IdP with admin console, federation, and OIDC, and can run a JVM service.
- bshaffer/oauth2-server-php — use instead for legacy PHP projects; older and less actively developed, but the long-standing alternative.
- steverhoades/oauth2-openid-connect-server — use alongside this library when you need OpenID Connect on top of its OAuth 2.0 core.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2012–2013 | Initial release; created by Alex Bilbie from the Linkey project[^1]. |
| 8.0.0 | 2019-07-13 | Moved onto `lcobucci/jwt`; dropped older PHP versions. |
| 8.2.0 | 2020-11-25 | Continued 8.x line; PSR-7 and grant refinements. |
| 9.0.0 | 2024-05-13 | Requires PHP 8.1+; JWT dependency major bump[^2]. |
| 9.3.0 | 2025-11-25 | Feature release on the 9.x line. |
| 9.4.1 | 2026-06-25 | Latest release; PHP 8.2–8.5 support. |

## References

[^1]: README — credits and maintenance history (Alex Bilbie 2012–2017, Andy Millington maintainer; Mozilla Secure Open Source Fund audit). https://github.com/thephpleague/oauth2-server
[^2]: Official documentation — supported grants, RFCs, and community integrations. https://oauth2.thephpleague.com
[^3]: PSR-7: HTTP Message Interface, PHP-FIG. https://www.php-fig.org/psr/psr-7/
[^4]: OAuth 2.0 Security Best Current Practice / OAuth 2.1 draft — deprecation of the implicit and password grants. https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics

## Tags

php, oauth2, authorization-server, authentication, security, jwt, psr-7, api-security, identity, library
