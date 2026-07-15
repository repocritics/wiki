# briansmith/ring

> Safe, fast, small-surface cryptographic primitives for Rust — a curated Rust API over BoringSSL's C and assembly.

[GitHub repo](https://github.com/briansmith/ring) ·
[docs.rs](https://docs.rs/ring) ·
[License: ISC-style + BoringSSL/OpenSSL (mixed)](https://github.com/briansmith/ring/blob/main/LICENSE)

## Overview

`ring` is a Rust cryptography library that exposes a deliberately small set of modern primitives: digests (SHA-2 family, SHA-1 for legacy), HMAC, HKDF, PBKDF2, AEAD (AES-GCM, ChaCha20-Poly1305), key agreement (X25519, ECDH over P-256/P-384), and signatures (Ed25519, ECDSA P-256/P-384, RSA PKCS#1 and PSS)[^1]. Most of the actual math is not written in Rust — the C and assembly come from BoringSSL, itself a fork of OpenSSL, with some formally-verified field arithmetic from fiat-crypto. `ring` wraps that code in a memory-safe, misuse-resistant Rust API.

The project's own framing matters: the maintainer, Brian Smith, describes `ring` as "an experiment" and has never shipped a 1.0[^2]. The README quotes BoringSSL's own warning against third-party use and passes it through. Despite that posture, `ring` became the de facto crypto backend of the Rust TLS ecosystem — for years it was the only crypto provider for `rustls`, and it still underpins large parts of `quinn`, `tokio-rustls`, and many networked Rust services.

The defining tension is exactly that gap: a library disclaimed as an experiment, maintained largely by one person, sitting under a very large fraction of production Rust TLS traffic. That single-maintainer bus factor and a historically slow, opaque release cadence are the reasons `rustls` later adopted `aws-lc-rs` as an alternative (and now default) provider[^3].

## Getting Started

```bash
cargo add ring
```

`ring`'s build compiles bundled C and assembly, so a working C toolchain (Clang recommended, matching what BoringSSL uses) must be present at build time.

```rust
use ring::{rand, signature::{self, KeyPair}};

fn main() -> Result<(), ring::error::Unspecified> {
    // Generate an Ed25519 key pair, serialized as PKCS#8.
    let rng = rand::SystemRandom::new();
    let pkcs8 = signature::Ed25519KeyPair::generate_pkcs8(&rng)?;
    let key_pair = signature::Ed25519KeyPair::from_pkcs8(pkcs8.as_ref())?;

    let message = b"hello, ring";
    let sig = key_pair.sign(message);

    // Verify with only the public key.
    let public_key = signature::UnparsedPublicKey::new(
        &signature::ED25519,
        key_pair.public_key().as_ref(),
    );
    public_key.verify(message, sig.as_ref())?;
    Ok(())
}
```

The API is uniform in one respect that trips newcomers: nearly every fallible operation returns `ring::error::Unspecified` — a deliberately opaque error type that reveals nothing about *why* an operation failed, to avoid leaking oracle information.

## Architecture / How It Works

`ring` is a three-layer stack, not a pure-Rust library:

1. **C / assembly core** — vendored from BoringSSL. Per-architecture assembly (generated via perlasm) provides constant-time, hardware-accelerated implementations (AES-NI, AVX2, ARM crypto extensions). Some elliptic-curve field arithmetic is imported from fiat-crypto, which is machine-generated from formally-verified specifications.
2. **`build.rs`** — the source of most operational friction. At build time it selects and compiles the correct pre-generated assembly for the target and links the C code via the `cc` crate. `ring` only ships assembly for a fixed set of supported targets; anything outside that set is a problem (see Production Notes).
3. **Rust API** — the public modules (`aead`, `agreement`, `digest`, `hkdf`, `hmac`, `pbkdf2`, `rand`, `signature`, `rsa`, `constant_time`, `pkcs8`, `error`). This layer is where the "safe and misuse-resistant" design lives: nonce types that are hard to reuse, sealed algorithm identifiers (`&AES_256_GCM`, `&ED25519`), and no general-purpose "raw block cipher" escape hatches.

The design philosophy is *subtractive*. There is no MD5, no DES, no ECB, no low-level cipher API you can misuse; the algorithm menu is curated to what a modern protocol should use. This is a feature for application authors and a limitation for anyone who needs an exotic or legacy primitive.

Because randomness comes from `SystemRandom` (the OS CSPRNG) and algorithm choices are constants, most `ring` code has very little room for the classic footguns — parameter confusion, nonce misuse across a key, or accidental weak-algorithm selection.

## Production Notes

**Cross-compilation is the number one pain point.** Because `ring` compiles bundled assembly for a fixed target list, cross-compiling to a target `ring` does not explicitly support (some `musl`, some embedded, some Windows/ARM, WASM variants over time) can fail outright or require environment plumbing: a correct `CC`, `AR`, `TARGET_CC`, and sometimes a specific Clang. CI matrices that "just work" on x86-64 Linux frequently break the first time they add a new target. Teams that hit this repeatedly are a large part of why `aws-lc-rs` exists as a drop-in alternative.

**Single-maintainer, opaque cadence.** `ring` is effectively one maintainer's project. Releases have historically been infrequent and lightly documented; the README explicitly recommends reviewing *every commit* rather than relying on curated release notes[^2]. For a dependency this deep in the stack, that is a real supply-chain and continuity consideration, not a stylistic complaint.

**Security advisories have happened.** `ring` has been the subject of RustSec advisories — including a period where older `0.16.x` was flagged as unmaintained, and later a panic/overflow issue in AES handling fixed in a `0.17.x` release. Pin a current `0.17` line and watch `cargo audit`; do not stay on `0.16`.

**Side-channel caveats are explicit.** `ring` ships a `SIDE-CHANNELS.md` documenting the *limits* of its constant-time guarantees — they depend on the compiler and target behaving as expected, and are not absolute across all toolchains and microarchitectures[^4]. Treat unusual toolchains/targets as unvetted.

**Version churn in the ecosystem.** `rustls` moving to a pluggable `CryptoProvider` model means "which crypto backend" is now a real decision. Mixing crates that each pin a different provider, or upgrading across the `ring` `0.16 → 0.17` boundary (which changed some APIs and MSRV), is a common source of dependency-resolution and build breakage.

## When to Use / When Not

**Use when:**
- You need vetted, hardware-accelerated primitives with a hard-to-misuse Rust API and don't want to hand-assemble RustCrypto components.
- You're building on `rustls`/`quinn` and want the long-established, widely-deployed provider.
- Your build targets are mainstream (x86-64 / aarch64 Linux, macOS, Windows) where the bundled assembly is well-trodden.

**Avoid when:**
- You cross-compile to unusual or embedded targets, or need pure-Rust with no C toolchain — the `build.rs` will fight you.
- You need FIPS certification or a corporate-maintained backend with SLAs — prefer `aws-lc-rs`.
- You need legacy or exotic algorithms `ring` deliberately omits.
- Single-maintainer supply-chain risk is unacceptable for your threat model.

## Alternatives

- aws/aws-lc-rs — Rust bindings to AWS-LC (another BoringSSL derivative); corporate-maintained, FIPS-capable, now the default `rustls` provider. Use when you need cross-compile reliability, FIPS, or a maintained backend.
- RustCrypto/* (hashes, AEADs, signatures, elliptic-curves) — pure-Rust, no C toolchain, broad algorithm coverage. Use when you must avoid C/asm or need an algorithm `ring` omits.
- sfackler/rust-openssl — bindings to system OpenSSL; full-featured and battle-tested. Use when you need OpenSSL parity or must link the platform's OpenSSL.
- dalek-cryptography/curve25519-dalek — pure-Rust X25519/Ed25519 building blocks. Use when you only need Curve25519 and want a pure-Rust dependency.
- orion-rs/orion — pure-Rust, audited, misuse-resistant high-level API. Use when you want `ring`'s safety posture without any C.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015-08 | First crates.io publish; shared as "an experiment"[^2]. |
| 0.13 | 2018 | Long-lived early series; API consolidation. |
| 0.16.0 | 2019-08 | Widely-adopted line; `rustls` era backend. |
| 0.17.0 | 2023-11 | API/MSRV changes, updated BoringSSL/asm; current major line. |

`ring` has intentionally never reached 1.0 — the `0.x` versioning reflects the "experiment" framing, not immaturity of the code itself.

## References

[^1]: `ring` API documentation. https://docs.rs/ring/latest/ring/
[^2]: `ring` README — "This project was originally shared on GitHub in 2015 as an experiment... It is an experiment." https://github.com/briansmith/ring/blob/main/README.md
[^3]: `rustls` `CryptoProvider` documentation (ring vs aws-lc-rs). https://docs.rs/rustls/latest/rustls/crypto/index.html
[^4]: `ring` SIDE-CHANNELS.md. https://github.com/briansmith/ring/blob/main/SIDE-CHANNELS.md

## Tags

rust, cryptography, security, tls, aead, digital-signatures, boringssl, ffi, assembly, crypto-primitives, constant-time
