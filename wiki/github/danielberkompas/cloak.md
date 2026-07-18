# danielberkompas/cloak

> Application-layer encryption for Elixir — tagged ciphertexts and vault modules, designed to pair with Ecto for encrypted database fields.

[GitHub repo](https://github.com/danielberkompas/cloak) ·
[Hex docs](https://hexdocs.pm/cloak/) ·
[License: MIT](https://github.com/danielberkompas/cloak/blob/master/LICENSE.md)

## Overview

Cloak is an Elixir library for symmetric encryption of application data, started by Daniel Berkompas in 2015. Its niche is field-level encryption: encrypting individual values (SSNs, tokens, PII columns) before they reach the database, rather than transport (TLS) or disk (LUKS/TDE) encryption. Three design choices define it: random IVs generated per encryption via `:crypto.strong_rand_bytes`, ciphertexts tagged with metadata identifying the cipher and key that produced them, and configuration through ordinary Elixir app config rather than external key files[^1].

The tagged-ciphertext design is the library's real contribution. Because every ciphertext self-describes which cipher/key version encrypted it, a vault can hold several keys at once and route each decryption to the right one — turning key rotation from an all-at-once event into an incremental re-encryption. The tradeoff is a proprietary envelope format: Cloak ciphertexts are not raw AES output and are not readable by other tools without reimplementing the framing.

Since version 1.0 the repo contains only the encryption core; the Ecto types that made Cloak popular live in the companion package `cloak_ecto`[^2]. At 624 stars it is small in absolute numbers but the de facto standard for this job in Elixir — most "encrypt an Ecto field" search results end here. Maintenance is sporadic-but-alive: releases cluster in bursts (1.1.1–1.1.4 landed on a single day in April 2024), with the last commit in March 2026.

## Getting Started

```elixir
# mix.exs
{:cloak, "~> 1.1"}          # add {:cloak_ecto, "~> 1.3"} for Ecto fields
```

Define a vault and add it to your supervision tree:

```elixir
defmodule MyApp.Vault do
  use Cloak.Vault, otp_app: :my_app
end

# config/runtime.exs — never commit keys; decode from env at boot
config :my_app, MyApp.Vault,
  ciphers: [
    default: {Cloak.Ciphers.AES.GCM,
      tag: "AES.GCM.V1",
      key: Base.decode64!(System.fetch_env!("CLOAK_KEY")),
      iv_length: 12}
  ]
```

```elixir
{:ok, ciphertext} = MyApp.Vault.encrypt("plaintext")
{:ok, "plaintext"} = MyApp.Vault.decrypt(ciphertext)
```

## Architecture / How It Works

A vault is a module you define with `use Cloak.Vault`, started under your supervision tree. At startup it reads its cipher config (from app config or an `init/1` callback) and caches it in an ETS table, so per-call encryption does not round-trip through a process — vaults are not a bottleneck on the hot path. Multiple vaults can run simultaneously, which maps naturally onto umbrella apps or per-tenant key separation.

Ciphers implement the `Cloak.Cipher` behaviour (`encrypt/2`, `decrypt/2`, `can_decrypt?/2`). The library ships AES-256-GCM and AES-CTR implementations on top of Erlang's `:crypto`; anything else (ChaCha20, KMS-backed ciphers) is a behaviour implementation away. Encryption always uses the first cipher in the configured list; decryption asks each cipher `can_decrypt?/2` and routes to the one whose tag matches.

The wire format prepends a length-prefixed tag to the IV and ciphertext — the README's example bytes `<<1, 10, "AES.GCM.V1", ...>>` show a format byte, tag length, then the tag string[^1]. This enables multi-key decryption, and also makes the output Cloak-specific.

Key rotation is the intended workflow: add a new cipher at the head of the list (new tag, new key), keep the old cipher below it, then re-encrypt existing data in batches — `cloak_ecto` documents the migration pattern[^2]. Old ciphertexts keep decrypting throughout.

## Production Notes

- **Encrypted fields are opaque to SQL.** No `WHERE email = ?`, no `LIKE`, no ordering, no indexes on plaintext semantics. `cloak_ecto` provides hashed companion fields (HMAC/SHA-256 types) for exact-match lookup — a blind-index pattern you must design into the schema up front, not bolt on later.
- **The GCM IV-length footgun.** Cloak historically used 16-byte IVs for AES-GCM; the ecosystem standard is 12 bytes. Issue #93 covers the interoperability fallout, and the README tells you to pass `iv_length: 12` explicitly[^3]. The comment promising a fixed default "in Cloak 2.0" has been there for years — 2.0 has never shipped, so the flag remains your responsibility.
- **Threat model honesty.** Keys live in application memory and app config. Cloak protects against database dumps, stolen backups, and DBA-level snooping; it does nothing against a compromised application node. If keys must never touch the app, you want an external KMS/transit service instead.
- **Ciphertext expands.** IV + tag metadata + GCM auth tag add tens of bytes per value on top of the encrypted payload; columns must be `binary`/`bytea` (or Base64-encoded text at ~33% further overhead). Budget for it on wide tables.
- **Rotation is a table rewrite.** Tagged ciphertexts make rotation *safe*, not *free* — re-encrypting means updating every row, with the usual long-migration concerns (locks, replication lag) on large tables. And since vault config is cached at startup, runtime key changes require re-saving the config or restarting the vault process.
- **Release cadence.** Years can pass between releases (1.1.0 in 2021, patches in 2024). The surface is small and sits on OTP's `:crypto`, so this is less alarming than for a larger framework — but expect slow issue turnaround (11 open issues).

## When to Use / When Not

**Use when:**

- You need field-level encryption of PII in an Elixir/Phoenix app backed by Ecto — this is the paved road (`cloak` + `cloak_ecto`).
- You anticipate key rotation and want ciphertexts that self-identify their key version.
- Compliance requires "encryption at rest" at the application layer beyond what disk/volume encryption gives you.

**Avoid when:**

- You need to search, sort, or range-query the encrypted values — blind indexes only buy exact match; anything richer needs a different design.
- Your threat model includes application-server compromise; use an external KMS (Vault transit, AWS KMS) so keys never reside in app memory.
- You need cross-language ciphertext interop — the tagged envelope is Cloak-specific.
- You are not using Ecto and just need primitives — `:crypto` or libsodium bindings are less machinery.

## Alternatives

- danielberkompas/cloak_ecto — not an alternative but the required companion for Ecto field types; most users install both.
- jlouis/enacl — libsodium (NaCl) bindings; use instead when you want modern primitives (XChaCha20-Poly1305, sealed boxes) and control of the format.
- ntrepid8/ex_crypto — thin wrapper over Erlang `:crypto`; use for one-off encryption without vault/rotation machinery.
- hashicorp/vault — transit secrets engine does encrypt/decrypt over the network; use when keys must never live in application memory.
- postgres/postgres — `pgcrypto` encrypts inside the database; use when you prefer DB-side encryption and accept keys appearing in SQL statements.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2015-09 | Initial release. |
| 0.7.0 | 2018-08 | Deprecation groundwork for the 1.0 config model. |
| 0.9.0 | 2018-09 | Last 0.x line; migration guide to 1.0 published[^4]. |
| 1.0.0-alpha.0 | 2018-12 | Vault-based redesign; Ecto types split into `cloak_ecto`[^2]. |
| 1.1.0 | 2021-06 | Maintenance release. |
| 1.1.1–1.1.4 | 2024-04 | Patch burst, all published the same day. |

## References

[^1]: Cloak README. https://github.com/danielberkompas/cloak/blob/master/README.md
[^2]: `cloak_ecto` on Hex — Ecto types for Cloak vaults. https://hex.pm/packages/cloak_ecto
[^3]: Issue #93 — AES-GCM IV length interoperability. https://github.com/danielberkompas/cloak/issues/93
[^4]: Upgrade guide, "How to upgrade from Cloak 0.9.x to 1.0.x". https://hexdocs.pm/cloak/0-9-x_to_1-0-x.html

## Tags

elixir, encryption, cryptography, ecto, field-level-encryption, key-rotation, aes-gcm, security, pii, hex-package, library
