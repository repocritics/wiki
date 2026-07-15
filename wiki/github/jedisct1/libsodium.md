# jedisct1/libsodium

> A portable, packageable fork of NaCl that turns modern cryptographic primitives into a small, misuse-resistant C API.

[GitHub repo](https://github.com/jedisct1/libsodium) ·
[Official website](https://libsodium.org) ·
[License: ISC](https://github.com/jedisct1/libsodium/blob/master/LICENSE)

## Overview

libsodium is a cryptography library written in C, started by Frank Denis (jedisct1) in 2013 as a portable fork of Daniel J. Bernstein's NaCl[^1]. NaCl was a research-grade library with excellent primitives but a build system and API that resisted packaging; libsodium keeps NaCl's API surface, adds sensible high-level wrappers, and makes the whole thing cross-compilable, installable, and distributable through OS package managers. As of 2026 it has ~13.8k stars and is the crypto backend behind PHP's core `sodium` extension, Signal-adjacent tooling, WireGuard-era key handling, and language bindings in nearly every ecosystem[^2].

Its defining design choice is *opinionation*. Where OpenSSL exposes hundreds of algorithms and configuration knobs, libsodium exposes a curated set of primitives — Curve25519, Ed25519, XSalsa20/ChaCha20, Poly1305, BLAKE2b, Argon2 — behind APIs that are hard to misuse. There is almost no algorithm agility: you get one good default per task. That is the point. The tradeoff is that libsodium is deliberately *not* a protocol library — it has no TLS, no X.509, no ASN.1. It gives you authenticated encryption and signatures; you (or a higher layer) build the protocol.

The library is not FIPS 140 validated, which rules it out of some regulated environments regardless of the quality of the primitives[^3].

## Getting Started

```bash
# Debian / Ubuntu
apt install libsodium-dev
# macOS
brew install libsodium
# From source (releases are signed with minisign)
./configure && make && make check && sudo make install
```

```c
#include <sodium.h>
#include <string.h>

int main(void) {
    if (sodium_init() < 0) {
        return 1;               /* library could not be initialized */
    }

    unsigned char key[crypto_secretbox_KEYBYTES];
    unsigned char nonce[crypto_secretbox_NONCEBYTES];
    const unsigned char msg[] = "hello";
    unsigned char ct[crypto_secretbox_MACBYTES + sizeof msg - 1];

    crypto_secretbox_keygen(key);
    randombytes_buf(nonce, sizeof nonce);           /* 24-byte nonce: random is safe */
    crypto_secretbox_easy(ct, msg, sizeof msg - 1, nonce, key);
    /* transmit nonce + ct together; nonce is not secret */
    return 0;
}
```

Compile with `cc app.c -lsodium`. `sodium_init()` must be called once, before any other function and before spawning threads.

## Architecture / How It Works

libsodium is organized as a set of primitive families, each with a low-level API (named after the exact algorithm) and a high-level "easy" API with defaults:

- **`crypto_secretbox`** — secret-key authenticated encryption. XSalsa20 stream cipher + Poly1305 MAC. 24-byte nonces mean random nonces are collision-safe.
- **`crypto_box`** — public-key authenticated encryption. X25519 key agreement feeding the same XSalsa20-Poly1305 construction.
- **`crypto_sign`** — Ed25519 detached and attached signatures.
- **`crypto_aead`** — AEAD constructions: XChaCha20-Poly1305 (192-bit nonce, random-safe), ChaCha20-Poly1305 IETF, and AES-256-GCM (96-bit nonce, requires counter discipline).
- **`crypto_generichash`** — BLAKE2b, keyed or unkeyed, arbitrary output length.
- **`crypto_pwhash`** — password hashing / key derivation, Argon2id by default (scrypt available).
- **`crypto_secretstream`** — chunked stream/file encryption (XChaCha20-Poly1305) with rekeying and per-chunk tags.
- **`crypto_kx`**, **`crypto_kdf`**, **`randombytes`** — key exchange, subkey derivation, and a CSPRNG that defers to the OS entropy source.

Implementations are constant-time by construction, and at runtime the library detects CPU features (AES-NI, AVX2, NEON) and dispatches to optimized code paths while producing byte-identical results across platforms. This portability extends to WebAssembly via the emscripten-built `libsodium.js`, iOS, Android, and MinGW/Visual Studio on Windows[^2]. A `build.zig` ships in-tree, hence the `zig-package` topic.

Secure-memory helpers are a distinguishing feature: `sodium_malloc` returns page-aligned allocations flanked by guard pages and a canary, `sodium_mlock` keeps secrets out of swap, and `sodium_memzero` wipes buffers without the compiler optimizing the write away. `sodium_memcmp` provides constant-time comparison — using `memcmp` on MACs or tags is a classic timing leak.

## Production Notes

**Nonce reuse is the primary footgun.** For the 24-byte-nonce primitives (`secretbox`, `box`, XChaCha20) random nonces are safe. But AES-256-GCM and the IETF ChaCha20-Poly1305 variant use 96-bit nonces where random generation risks birthday-bound collisions; those demand a counter or dedicated nonce-management scheme. Reusing a `(key, nonce)` pair is catastrophic for confidentiality and, for GCM/Poly1305, forgeability.

**`crypto_pwhash` parameters are a DoS surface.** `OPSLIMIT`/`MEMLIMIT` scale CPU and memory cost; the `SENSITIVE` presets can require on the order of a gigabyte of RAM per hash. If an attacker controls the cost parameters or can trigger many concurrent hashes, memory exhaustion is the failure mode. Pin server-side parameters; do not derive them from client input.

**Initialization ordering.** `sodium_init()` returns `0` on success, `1` if already initialized, `-1` on failure. Modern versions are safe to call it more than once, but it must complete before any multithreaded use — the internal CPU-feature and RNG setup is not itself synchronized against concurrent first-use.

**Versioning has long gaps by design.** Point release 1.0.18 (2019) served as the de facto stable release for roughly five years before 1.0.19 (2024); throughout that span the `stable` branch received continuous security and maintenance updates that were ABI-compatible with the parent point release[^4]. The practical rule: pin to a point release for ABI, and apply `stable`-branch updates freely. Check `sodium_version_string()` and the `SODIUM_LIBRARY_VERSION_*` macros; a library's SONAME encodes ABI compatibility, not the marketing version.

**No TLS, no protocol.** libsodium gives you primitives. If you need a wire protocol, use `crypto_secretstream` for framing, a Noise implementation for handshakes, or a real TLS library. Building an ad-hoc protocol directly on `crypto_box` is where most integration bugs live.

**Verify release integrity.** Tarballs are signed with minisign (Frank Denis's own signing tool) and GPG; distributions that skip signature verification have historically been a supply-chain gap.

## When to Use / When Not

**Use when:**
- You need modern authenticated encryption or signatures and want an API that steers you away from mistakes.
- You are embedding crypto into an application (desktop, mobile, WASM) and want one portable dependency.
- You want secure-memory primitives (guard pages, mlock, constant-time compare) out of the box.
- You are writing bindings for a higher-level language and want a stable C ABI.

**Avoid when:**
- You need TLS, X.509, or a full protocol stack — reach for a TLS library instead.
- You are in a FIPS 140-mandated environment — libsodium is not validated[^3].
- You need broad algorithm agility or legacy-algorithm interop (RSA, DES, specific curves libsodium omits).
- You are on a severely constrained MCU where even libsodium's footprint is too large — see libhydrogen.

## Alternatives

- jedisct1/libhydrogen — same author's smaller library for embedded/constrained targets; fewer primitives, tiny footprint. Use when libsodium is too heavy for the MCU.
- openssl/openssl — full TLS + certificate + broad-algorithm toolkit. Use when you need protocols, agility, or a FIPS module, and can absorb the larger API footgun surface.
- google/boringssl — OpenSSL fork with no stable API. Use only when vendoring it inside a project you fully control.
- tink-crypto/tink — Google's multi-language misuse-resistant library with key management. Use when you want higher-level key handling across several languages.
- LoupVaillant/Monocypher — single-file C implementation of overlapping primitives. Use when you want to vendor one auditable source file with no build system.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013-01 | Repository created; fork of NaCl begins[^1]. |
| 1.0.18 | 2019-05 | Long-lived point release; de facto stable for ~5 years. |
| 1.0.19 | 2024 | First point release after the ~5-year gap[^4]. |
| 1.0.20 | 2024 | Maintenance point release on the 1.0.x line. |

## References

[^1]: NaCl: Networking and Cryptography library (Bernstein, Lange, Schwabe). https://nacl.cr.yp.to/
[^2]: libsodium documentation — usage, bindings, and platform support. https://doc.libsodium.org/
[^3]: libsodium FAQ on FIPS 140 and standards positioning. https://doc.libsodium.org/quickstart
[^4]: libsodium versioning policy and release stream (README, `stable` branch). https://github.com/jedisct1/libsodium/blob/master/README.md

## Tags

c, cryptography, encryption, digital-signatures, nacl, authenticated-encryption, curve25519, ed25519, password-hashing, security, portable, zig-package
