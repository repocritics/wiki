# marshallpierce/rust-base64

> The de facto base64 implementation for Rust — correctness-first, `no_std`-capable, and published to crates.io as `base64`.

[GitHub repo](https://github.com/marshallpierce/rust-base64) ·
[crates.io](https://crates.io/crates/base64) ·
[docs.rs](https://docs.rs/base64) ·
License: MIT OR Apache-2.0

## Overview

`rust-base64` is the crate that publishes as `base64`, the near-universal choice for Base64 encoding and decoding in the Rust ecosystem[^1]. The repository dates to 2015 and has accreted through the language's own maturation; it forbids `unsafe` (the `unsafe-forbidden` safety-dance badge), is fuzzed via `cargo-fuzz`, and states two goals plainly: be *correct* and be *fast*. For a primitive that sits under TLS certificates, JWTs, data URIs, and MIME payloads, "correct" is the load-bearing word — the maintainer notes that PRs are scrutinized precisely because the crate is used in security-sensitive contexts.

The defining tension is API surface versus convenience. Base64 looks trivial, but the crate deliberately exposes several abstraction levels so callers can trade allocation for speed: `encode`/`decode` return a fresh `Vec`/`String`, while `encode_slice`/`decode_slice` write into caller-owned buffers and avoid allocation entirely. Layered on top is a matrix of *alphabets* (standard vs. URL-safe) and *padding policies* (required, forbidden, indifferent). Getting Base64 right in the wild means confronting non-canonical encodings — stray padding, trailing bits set to 1 — and the crate makes those decisions explicit rather than silently lenient.

The cost of that rigor is a genuinely disruptive API history. The `0.20`/`0.21` line replaced the old free-function-plus-`Config` interface with an `Engine` trait, and code written against `base64` 0.13 does not compile against 0.22 without a rewrite. This is the single largest source of friction for anyone upgrading an old dependency tree.

## Getting Started

```toml
# Cargo.toml
[dependencies]
base64 = "0.22"
```

```rust
use base64::prelude::*;

fn main() {
    let encoded = BASE64_STANDARD.encode(b"hello world");
    assert_eq!(encoded, "aGVsbG8gd29ybGQ=");

    let decoded = BASE64_STANDARD.decode(&encoded).unwrap();
    assert_eq!(decoded, b"hello world");

    // URL-safe alphabet, no padding — common for JWTs and query params.
    let token = BASE64_URL_SAFE_NO_PAD.encode(b"\xFF\xEE");
    assert_eq!(token, "_-4");
}
```

The `Engine` trait must be in scope to call `.encode()` / `.decode()`; the `prelude` brings it in along with the standard engine constants. Without the prelude, import it explicitly: `use base64::{engine::general_purpose::STANDARD, Engine as _};`.

## Architecture / How It Works

Since the 0.21 rework, everything routes through the `Engine` trait[^2]. An engine bundles three things: an **alphabet** (the 64-character mapping plus which two characters fill slots 62–63), a **`Config`** (encode padding on/off, decode padding mode, whether to tolerate non-zero trailing bits), and the encode/decode logic itself.

- **`GeneralPurpose`** is the concrete engine nearly everyone uses. The crate ships four ready-made `const` instances in `engine::general_purpose`: `STANDARD`, `STANDARD_NO_PAD`, `URL_SAFE`, `URL_SAFE_NO_PAD`. The `prelude` re-exports them as `BASE64_STANDARD` and friends.
- **`Alphabet`** is a 64-byte table. `alphabet::STANDARD` uses `+/`; `alphabet::URL_SAFE` uses `-_`. `Alphabet::new` builds a custom one at const-eval time (e.g. the bcrypt or crypt variants), rejecting duplicates and invalid bytes.
- **`GeneralPurposeConfig`** controls behavior. `DecodePaddingMode` has three settings — `Indifferent` (default, accept correct-or-absent padding), `RequireCanonical`, and `RequireNone` — plus `decode_allow_trailing_bits` for the sloppy-encoder case.

Decoding runs a table-driven hot loop: a 256-entry lookup array maps each input byte to its 6-bit value or a sentinel `INVALID_VALUE`, processing input in 8-byte chunks with a scalar tail. There is no SIMD path in the crate itself — the speed comes from tight branch-predictable scalar code, not vectorization. The encode path is symmetric.

Three I/O adapters exist for streaming: `read::DecoderReader` wraps a `Read` and decodes on the fly, `write::EncoderWriter` wraps a `Write` and encodes as bytes are written, and `write::EncoderStringWriter` accumulates into a `String`. These are gated behind the `std` feature.

## Production Notes

**The 0.13 → 0.21+ migration is the real cost of this crate.** Old code calls `base64::encode(x)` / `base64::decode(x)` / `encode_config(x, base64::URL_SAFE)`. Those free functions are gone. Every call site must be rewritten to `ENGINE.encode(x)` with the `Engine` trait imported, and the `Config` enum values map to different engine constants. Budget real time for this on any codebase that pinned an old version. Transitive dependencies frequently pull in multiple major versions of `base64` simultaneously — check `cargo tree -d`.

**Whitespace and interspersed junk are rejected, by design.** The decoder does not skip newlines, nulls, or spaces; MIME/PEM-style line-wrapped input will fail. You must strip non-alphabet bytes first (the README points at `iter-read` or `Vec::retain`), and use the separate `line-wrap` crate to *produce* wrapped output. This trips up people migrating from lenient decoders in other languages.

**Padding is a decision you must make, not a default to ignore.** Standard `=` padding vs. no-padding is a wire-format contract with the other side. JWTs and many URL-safe contexts use `NO_PAD`; misuse produces `DecodeError::InvalidPadding` or `InvalidLength`. `RequireCanonical` mode is what you want when you need to reject malleable encodings (relevant to signature and content-addressing schemes).

**Allocation control matters at volume.** `encode`/`decode` allocate per call. In hot paths, precompute `encoded_len`, allocate once, and use `encode_slice`/`decode_slice` into a reused buffer — this is the difference the README quotes between the ~2.1 GiB/s allocating path and the ~2.6 GiB/s slice path. Note that `decode_slice` returns a `DecodeSliceError` if the output buffer is too small.

**`no_std`.** Disable default features to target `core`. The `alloc` feature restores `Vec`/`String`-returning methods; without `std` you lose the `Read`/`Write` adapters and `std::error::Error` impls. The MSRV is Rust 1.48.0, which is conservative enough to slot into most embedded toolchains.

## When to Use / When Not

**Use when:**
- You need Base64 in any Rust project — this is the ecosystem default and the safe pick.
- You want explicit control over alphabet and padding, or a custom alphabet.
- You're in `no_std` / embedded and need a `core`-compatible codec.
- You need streaming encode/decode over `Read`/`Write` without buffering the whole payload.

**Avoid / look elsewhere when:**
- You need lenient decoding that skips whitespace transparently — this crate makes you strip input yourself first.
- You want SIMD-accelerated throughput beyond what scalar code delivers; a specialized vectorized codec may edge it out on huge payloads.
- You only ever handle one fixed variant and want zero API ceremony — the `Engine` trait is a small tax, but it is a tax.

## Alternatives

- rustcrypto/formats (`base64ct`) — constant-time, `no_std`, no heap; use it when you're decoding secrets and want data-independent timing.
- marshallpierce/rust-base64 itself is the fallback for almost everything else; the alternatives are niche by comparison.
- data-encoding — use it when you need base32/base16/base64 and custom encodings behind one uniform, spec-driven API.
- Roll a fixed-table inline encoder — only justified in `no_std` firmware where you want zero dependencies and one hardcoded alphabet.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2015-12 | First release; repository created 2015-12-04. |
| 0.13.x | 2020–2021 | Long-lived classic API: free `encode`/`decode` + `encode_config`/`Config`. |
| 0.20.0 | 2022 | `Engine` trait introduced; config-based free functions deprecated[^2]. |
| 0.21.0 | 2023 | Engine API stabilized; old free functions removed — the hard break. |
| 0.22.x | 2024 | `prelude` with `BASE64_*` constants; current release line[^1]. |

## References

[^1]: `base64` on crates.io and docs.rs. https://crates.io/crates/base64 · https://docs.rs/base64
[^2]: `Engine` trait documentation, `base64::engine`. https://docs.rs/base64/latest/base64/engine/trait.Engine.html
[^3]: Repository README, "no_std", canonical-encoding FAQ, and Rust version compatibility. https://github.com/marshallpierce/rust-base64

## Tags

rust, base64, encoding, no-std, codec, cryptography-adjacent, library, crates-io, data-encoding
