# FiloSottile/age

> A file encryption tool and Go library with small explicit keys, no config knobs, and UNIX-pipe composability.

[GitHub repo](https://github.com/FiloSottile/age) ·
[Official website](https://age-encryption.org) ·
[License: BSD-3-Clause](https://github.com/FiloSottile/age/blob/main/LICENSE)

## Overview

age ("aged", spelled lowercase, hard *g*) is a modern file encryption tool, on-disk format, and Go library, designed by Ben Cartwright-Cox (@benjojo) and Filippo Valsorda (@FiloSottile) starting in 2019[^1]. Its thesis is a reaction to GnuPG: instead of a configurable, backward-compatible, do-everything cryptosystem, age exposes exactly one modern construction, no algorithm agility, and no options. Keys are short Bech32 strings (`age1...` recipients, `AGE-SECRET-KEY-1...` identities) that fit on one line and paste cleanly into configs.

The format is specified independently of the reference code at age-encryption.org/v1 and is now maintained under C2SP, the Community Cryptography Specification Project[^1]. That separation is deliberate: rage (Rust)[^5] and Typage (TypeScript) are interoperable implementations of the same bytes, so age is a format standard with a canonical Go implementation rather than a single tool.

The defining tradeoff is scope discipline. age encrypts and decrypts; it does **not** sign, and it carries no notion of sender authenticity, key revocation, web-of-trust, or keyservers. A decrypted age file proves only that the holder of an identity could open it — never who produced it. Teams that need authenticity pair age with a separate signing tool, and teams that need structured secrets layer SOPS on top. Everything outside "turn this file into ciphertext" is intentionally someone else's problem.

## Getting Started

```bash
brew install age          # macOS / Linux; also apt, dnf, pacman, winget, scoop
go install filippo.io/age/cmd/...@latest   # from source (module path is filippo.io/age)
```

```bash
# Generate a keypair; the public key is printed and the secret is written to key.txt
age-keygen -o key.txt
# Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p

# Encrypt a stream to a recipient, decrypt with the matching identity
tar cvz ~/data | age -r age1ql3z7...mcac8p > data.tar.gz.age
age --decrypt -i key.txt data.tar.gz.age > data.tar.gz

# Passphrase mode (age offers to autogenerate a strong one)
age -p secrets.txt > secrets.txt.age

# Encrypt to someone's GitHub SSH keys, no age key needed
curl https://github.com/benjojo.keys | age -R - photo.jpg > photo.jpg.age
```

```go
// Go library — filippo.io/age
recipient, _ := age.ParseX25519Recipient("age1ql3z7...mcac8p")
out, _ := os.Create("data.age")
w, _ := age.Encrypt(out, recipient)  // returns an io.WriteCloser
io.Copy(w, plaintext)
w.Close()                            // Close flushes the final STREAM chunk — do not skip
```

## Architecture / How It Works

An age file is a short text header followed by a binary payload. The header declares the format version, then one **stanza** per recipient. Encryption picks a random 128-bit *file key*; each stanza wraps that file key for one recipient, so a file encrypted to N recipients has N stanzas and any one identity can open it. The header ends with an HMAC-SHA-256 tag (keyed via HKDF from the file key) that binds all stanzas together, preventing recipient-list tampering.

Recipient types are pluggable at the stanza level:

- **X25519** (native `age1...` keys) — an ephemeral ECDH share plus HKDF-SHA-256 and a ChaCha20-Poly1305 wrap of the file key. This leaves no identifying tag, so ciphertext does not reveal which key it targets.
- **scrypt** (passphrase mode) — the file key is wrapped under a key stretched from the passphrase; there is exactly one scrypt stanza and it may not be combined with others.
- **ssh-ed25519 / ssh-rsa** — a convenience path that reuses existing SSH keys. It embeds a tag derived from the public key, which *does* let an observer link a file to a specific key, and RSA support pulls in heavier OAEP crypto[^6]. `ssh-agent` is not supported.

The payload is encrypted with ChaCha20-Poly1305 under the **STREAM** construction[^3]: the plaintext is split into 64 KiB chunks, each sealed with an incrementing counter nonce and a final-chunk marker. STREAM gives authenticated, resumable streaming without buffering the whole file, but decryption is strictly sequential — there is no random access, and the header must be fully read before any plaintext emerges. There is no compression (avoiding CRIME/BREACH-style oracles) and no algorithm negotiation.

Beyond built-in recipient types, age defines a **plugin protocol**: any `age-plugin-<name>` binary on `$PATH` speaks a documented stdin/stdout state-machine, letting hardware and KMS backends (`age-plugin-yubikey`, `age-plugin-tpm`, `age-plugin-se`) participate without changes to age itself.

## Production Notes

- **No authenticity, by design.** age never tells you who encrypted a file. If provenance matters (software releases, inbound files from untrusted parties), sign separately with minisign/signify. Do not read "it decrypted" as "it is genuine."
- **No revocation or rotation.** Removing a compromised recipient means re-encrypting every affected file with a new file key. There is no key server, expiry, or CRL. Track which files went to which keys yourself.
- **SSH recipients leak targeting metadata.** The embedded key tag makes files linkable to a public key; prefer native `age1` recipients when unlinkability matters[^6]. YubiKey-resident SSH keys cannot decrypt (only PIV via the plugin can), and `ssh-agent` is unavailable.
- **Passphrase strength is on you.** scrypt slows brute force but a weak human passphrase is still weak; the autogenerated passphrase exists because it is the safe default.
- **Post-quantum recipients are large.** Hybrid ML-KEM-768 + X25519 keys are supported in v1.3.0+ (recipients start `age1pq1...`), but a single recipient is roughly 2000 characters, which is awkward in YAML, env vars, or URLs[^4]. They are opt-in via `age-keygen -pq`.
- **Streaming caveats.** You can pipe terabytes, but you cannot seek. Any use case needing partial reads of a large ciphertext must decrypt from the start each time.
- **Library API discipline.** The Go module path is the vanity import `filippo.io/age`, not `github.com/...`. Forgetting to `Close()` the encrypt writer truncates the final authenticated chunk and produces an unopenable file.
- **Not a secrets manager.** For per-value secrets in Git, use SOPS with an age backend rather than hand-rolling; age alone encrypts whole files opaquely.

## When to Use / When Not

**Use when:**
- Encrypting backups, archives, or files to keys you control, or to a colleague's GitHub SSH keys.
- You want secrets-in-Git via SOPS/agenix/sops-nix with a simple, auditable primitive.
- You value short keys, zero configuration, and a spec with multiple interoperable implementations.
- You want a forward-looking, post-quantum-capable format without operating a PKI.

**Avoid when:**
- You need signing, sender authenticity, or non-repudiation (age does none of these).
- You require OpenPGP interoperability, keyservers, or web-of-trust for existing contacts.
- You need per-recipient revocation, key expiry, or random-access decryption of huge files.
- You are bound to FIPS-validated cryptography or must interoperate with an existing GPG workflow.

## Alternatives

- str4d/rage — memory-safe Rust implementation of the identical age format; use when you want a Rust binary/library or to avoid a Go toolchain.
- getsops/sops — structured secrets (YAML/JSON/env) with per-value encryption and KMS backends; use when you need Git-friendly secrets, and let it call age underneath.
- jedisct1/minisign — signatures, not encryption; use alongside age when you need authenticity/provenance that age deliberately omits.
- gpg (GnuPG) — use when you must interoperate with OpenPGP, email encryption, keyservers, or existing web-of-trust.
- google/tink — use when you are embedding encryption inside an application across languages and want managed key primitives rather than a file tool.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial design | 2019 | Format and tool designed by @benjojo and @FiloSottile; repo created 2019-05[^1]. |
| beta series | 2019–2021 | `age1` Bech32 keys, STREAM payload, X25519/scrypt/SSH recipients stabilized. |
| v1.0.0 | 2021 | First stable release; format frozen at `age-encryption.org/v1`. |
| v1.1.0 | 2023 | Plugin ecosystem and packaging maturation. |
| v1.2.0 | 2024 | Continued stability; broad distro packaging. |
| v1.3.0 | 2026 | Hybrid post-quantum ML-KEM-768 + X25519 recipients; `age-inspect` metadata tool[^4]. |

## References

[^1]: age v1 specification, maintained under C2SP. https://age-encryption.org/v1
[^2]: age(1) man page and command reference. https://filippo.io/age/age.1
[^3]: Hoang, Reyhanitabar, Rogaway, Shrimpton, "Online Authenticated-Encryption and its Nonce-Reuse Misuse-Resistance" (the STREAM construction used for the payload). https://eprint.iacr.org/2015/189
[^4]: Post-quantum key support and `age-inspect`, per the project README (v1.3.0+). https://github.com/FiloSottile/age#post-quantum-keys
[^5]: rage, the interoperable Rust implementation. https://github.com/str4d/rage
[^6]: SSH recipient tracking caveat, per the project README "SSH keys" section. https://github.com/FiloSottile/age#ssh-keys
[^7]: Go library reference. https://pkg.go.dev/filippo.io/age

## Tags

go, encryption, cryptography, cli, file-encryption, x25519, chacha20-poly1305, post-quantum, ssh-keys, secrets-management, unix-philosophy
