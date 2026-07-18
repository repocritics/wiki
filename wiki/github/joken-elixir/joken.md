# joken-elixir/joken

> The de facto JWT library for Elixir — a claims/validation layer over erlang-jose, not a cryptography implementation of its own.

[GitHub repo](https://github.com/joken-elixir/joken) ·
[Hex docs](https://hexdocs.pm/joken/) ·
[License: Apache-2.0](https://github.com/joken-elixir/joken/blob/main/LICENSE)

## Overview

Joken is the standard JSON Web Token (JWT) library in the Elixir ecosystem (started 2014 by Bryan Joseph). It signs and verifies JWS tokens and manages claim generation and validation. All actual cryptography is delegated to `erlang-jose` by Andrew Bennett (@potatosalad)[^1] — Joken's own value is the Elixir-idiomatic layer on top: a `use Joken.Config` macro that generates signing/verification functions, a claims pipeline with per-claim generate and validate hooks, and configuration via the standard `config.exs` mechanism.

The defining tradeoff is scope. Joken deliberately stops at "token in, claims out": no Plug integration, no session handling, no permission model, no revocation store. Teams either wire it into their own auth plumbing (common in API-only Phoenix apps) or reach for a fuller framework like Guardian. Version 2.0 (2019) was a breaking rewrite — the 1.x fluent pipeline (`token() |> with_claim(...)`) was replaced with module-based configuration, and signers moved from per-call arguments to named config entries[^2].

At 813 stars the repo looks modest, but stars undercount BEAM libraries; Hex downloads and its position as the JWT dependency in most Elixir API stacks are the better signal. The push cadence (last push 2026-02, single-digit open issues) is that of a finished library: it changes when OTP or `jose` moves, not because features are missing.

## Getting Started

```elixir
# mix.exs
def deps do
  [
    {:joken, "~> 2.6"},
    {:jason, "~> 1.4"}   # recommended JSON adapter
  ]
end
```

```elixir
# config/config.exs
config :joken, default_signer: "my-256-bit-secret"

defmodule MyApp.Token do
  use Joken.Config

  @impl true
  def token_config do
    default_claims(default_exp: 60 * 60)          # exp, iat, nbf, iss, aud, jti
    |> add_claim("role", fn -> "user" end, &(&1 in ["admin", "user"]))
  end
end

{:ok, token, claims} = MyApp.Token.generate_and_sign()
{:ok, claims} = MyApp.Token.verify_and_validate(token)
```

## Architecture / How It Works

Joken is a thin coordination layer with three moving parts:

1. **`Joken.Signer`** — wraps `JOSE.JWK` + `JOSE.JWS` from erlang-jose. A signer pairs an algorithm (HS256/384/512, RS, PS, ES, EdDSA families — whatever `jose` supports on your OTP) with key material. Signers are usually declared in `config.exs` and parsed once; the actual sign/verify ops bottom out in OTP's `:crypto` NIFs via `jose`[^1].
2. **`Joken.Config`** — a macro that injects `generate_and_sign/2`, `verify_and_validate/2` (plus `!` variants) and a `token_config/0` callback into your module. `token_config` returns a map of `%Joken.Claim{}` structs, each carrying an optional generate function (used at signing) and validate function (used at verification). `default_claims/1` provides RFC 7519 registered claims with sane generators (`jti`, `iat`, `nbf`, `exp`).
3. **`Joken.Hooks`** — a behaviour with `before_*`/`after_*` callbacks around each pipeline stage (generate, sign, verify, validate). This is the extension seam: the companion `joken_jwks` package implements `before_verify` to resolve the signer from a remote JWKS endpoint by `kid`, with caching and re-fetch on unknown-key misses[^3].

A security-relevant design choice: the verifying signer is chosen by *your configuration*, never by the token's `alg` header. That closes the classic algorithm-confusion / `alg: none` class of JWT attacks by construction — a token signed with an unexpected algorithm simply fails verification.

Time is abstracted behind `Joken.CurrentTime`, so `exp`/`nbf` checks can be frozen or mocked in tests without sleeping or generating short-lived tokens.

## Production Notes

- **Validations only run for claims present in the token.** A token issued without `exp` sails through a config that validates `exp` — the check is skipped, not failed. Enforce required claims explicitly if your threat model needs them.
- **Default `iss`/`aud` are the string `"Joken"`.** `default_claims/0` is a demo default, not a production one — override issuer, audience, and expiration (default 2 hours) for any real deployment.
- **Parse PEM/JWK signers once.** `Joken.Signer.create/3` with a PEM does non-trivial parsing; doing it per request shows up in profiles. Use the named-signer config path or cache the struct (module attribute, `:persistent_term`). The repo ships benchmark scripts comparing Joken against raw `jose` calls.
- **Crypto performance depends on OTP.** `jose` falls back to pure-Erlang implementations when the OTP `:crypto` module lacks an algorithm (notably EdDSA on older OTP releases) — correct but much slower. Keep OTP current if you use non-HS algorithms.
- **No revocation story.** JWTs are stateless by design; logout/ban requires your own denylist keyed on `jti`. Joken gives you `jti` generation but no storage — that is on you.
- **1.x → 2.x migration is a rewrite**, not a version bump: API, config format, and defaults all changed[^2]. Any pre-2019 tutorial or Stack Overflow answer showing `Joken.token/0` pipelines is dead code.
- **Third-party tokens (Auth0, Keycloak, Entra):** don't hand-copy public keys; use `joken_jwks` so key rotation at the provider doesn't break verification[^3].

## When to Use / When Not

**Use when:**
- You need to issue or verify JWTs in an Elixir/Phoenix API and want to own the surrounding auth plumbing.
- You verify tokens minted elsewhere (OIDC providers, other services) — pair with `joken_jwks`.
- You want testable time-dependent claims (mockable clock) and per-claim validation logic.

**Avoid when:**
- You want batteries-included auth (Plug pipelines, permissions, sessions) — Guardian or Pow cover more ground.
- You need JWE (encrypted tokens) or low-level JWK manipulation — drop down to erlang-jose directly; Joken's surface is JWS + claims.
- Your tokens are same-app session state — Phoenix's built-in `Phoenix.Token` or plain server-side sessions are simpler and revocable.

## Alternatives

- ueberauth/guardian — use instead when you want a full token-based auth framework with Plug integration, permissions, and pipelines, not just JWT primitives.
- potatosalad/erlang-jose — use directly when you need JWE, JWK conversion, or JOSE operations beyond signed-token claims handling.
- danschultzer/pow — use instead when you actually want user auth with server-side sessions in Phoenix rather than stateless tokens.
- erlef/oidcc — use when tokens come from an OpenID Connect provider and you want certified protocol handling (discovery, JWKS, token exchange) rather than raw JWT verification.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2014-09 | Repo created; JWT library for Elixir, Apache-2.0. |
| 1.x | 2015–2018 | Fluent pipeline API (`token() \|> with_claim(...)`). |
| 2.0 | 2019-01 | Breaking rewrite: `Joken.Config` macro, hooks, named signers in config, mockable clock[^2]. |
| 2.6 | 2023 | Current stable line; README pins `~> 2.6`[^4]. |

## References

[^1]: erlang-jose — JSON Object Signing and Encryption for Erlang/Elixir, the crypto layer Joken delegates to. https://github.com/potatosalad/erlang-jose
[^2]: Joken guide, "Migrating from Joken 1.0". https://hexdocs.pm/joken/migration_from_1.html
[^3]: joken_jwks — JWKS signer strategy hook for Joken. https://github.com/joken-elixir/joken_jwks
[^4]: Joken changelog. https://hexdocs.pm/joken/changelog.html

## Tags

elixir, erlang-otp, jwt, json-web-token, authentication, security, cryptography, tokens, hex-package, library
