# rustls/rustls

> A memory-safe TLS 1.2/1.3 library in Rust that is secure by default and refuses to speak obsolete crypto.

[GitHub repo](https://github.com/rustls/rustls) ·
[Documentation](https://docs.rs/rustls/) ·
[License: Apache-2.0 OR MIT OR ISC](https://github.com/rustls/rustls/blob/main/LICENSE)

## Overview

Rustls is a TLS implementation written from scratch in Rust, started by Joe Birr-Pixton (@ctz) in 2016[^1]. Its thesis is that most TLS vulnerabilities are either memory-safety bugs (OpenSSL's Heartbleed lineage) or misconfiguration (accepting SSLv3, RC4, export ciphers). Rustls attacks both: the core forbids `unsafe` code, and it ships no obsolete protocol versions or cipher suites at all. There is no knob to enable TLS 1.0/1.1, no RC4, no static-RSA key exchange, no renegotiation, and no TLS compression — those code paths do not exist rather than being off by default[^2].

The library is developed under ISRG's Prossimo memory-safety initiative (the same foundation behind Let's Encrypt), which funds @ctz full-time[^3]. It is used in production across the Rust ecosystem and is a selectable TLS backend for curl and hyper-based stacks. The defining tension is conservatism versus interoperability: by removing legacy protocol support Rustls sheds a large attack surface, but it will simply fail to connect to old or misconfigured peers that a permissive OpenSSL build would tolerate.

Rustls is pre-1.0. The `0.x` line has taken semver-breaking changes at nearly every minor release (0.20 → 0.21 → 0.22 → 0.23 → 0.24), and each upgrade tends to require real migration work — this is the single biggest ongoing cost of adopting it.

## Getting Started

```toml
# Cargo.toml — 0.24+ ships crypto as a separate provider crate
[dependencies]
rustls = "0.24"
rustls-aws-lc-rs = "0.24"   # or rustls-ring for easier builds
webpki-roots = "1"
```

```rust
use std::sync::Arc;
use rustls::{ClientConfig, RootCertStore};

// Install a process-wide crypto provider once, before building any config.
rustls_aws_lc_rs::default_provider()
    .install_default()
    .expect("provider already installed");

let roots = RootCertStore {
    roots: webpki_roots::TLS_SERVER_ROOTS.to_vec(),
};

let config = ClientConfig::builder()
    .with_root_certificates(roots)
    .with_no_client_auth();

let config = Arc::new(config);
// Hand `config` + a server name to ClientConnection, then drive I/O yourself.
```

For async use, most projects wrap this with `tokio-rustls` rather than driving the connection state machine by hand.

## Architecture / How It Works

Rustls is **sans-I/O**. The core `ClientConnection` / `ServerConnection` types are pure protocol state machines: they never touch a socket. You feed ciphertext in with `read_tls`, call `process_new_packets`, and pull ciphertext out with `write_tls`; plaintext moves through the connection's reader/writer. This makes the library runtime-agnostic — the same core backs blocking `std::net`, `mio`, and `tokio` — but it means "just open a connection" requires either the `Stream` helper or an integration crate[^2].

Key structural pieces:

- **`CryptoProvider`** — since 0.22 all cryptographic primitives are pluggable behind this struct[^4]. Rustls itself contains no cipher implementations. Two first-party providers exist: `rustls-aws-lc-rs` (default, full feature set including post-quantum, but needs a C toolchain to build) and `rustls-ring` (easier to build, no post-quantum). Third-party providers wrap OpenSSL, BoringSSL, mbedTLS, SymCrypt, wolfCrypt, and RustCrypto.
- **Certificate validation** is handled by `rustls-webpki`, a separate no-`unsafe` X.509 path-building crate. Trust anchors come from `webpki-roots` (a vendored copy of Mozilla's root bundle) or `rustls-platform-verifier` (delegates to the OS trust store).
- **Config builders** use a typestate pattern: `ClientConfig::builder()` walks through required decisions (roots, then client-auth) so that an incompletely configured connection cannot be constructed.
- **TLS 1.3 first.** 1.3 is the primary target; 1.2 support is deliberately restricted to AEAD suites (AES-GCM, ChaCha20-Poly1305) with forward-secret key exchange.

Post-quantum hybrid key exchange (`X25519MLKEM768`) is available through the aws-lc-rs provider, making Rustls one of the earlier TLS libraries to ship a deployable PQ handshake.

## Production Notes

**The "no CryptoProvider" panic is the #1 footgun.** From 0.23/0.24 onward, if you neither enable a default-provider feature nor call `install_default()` (or pass a provider per-config), the first handshake panics with *"no process-level CryptoProvider available"*. This surprises people upgrading from older versions where a provider was implicit. Install one explicitly at startup.

**aws-lc-rs build friction.** The recommended provider compiles C and needs a working C compiler, CMake, and (on Windows) NASM. In restricted CI or exotic targets this fails; the usual escape hatch is switching to `rustls-ring`, at the cost of losing post-quantum support and some suites. This build-vs-features tradeoff is the most common provider decision.

**Interoperability failures are by design.** Rustls will not negotiate with peers that only offer TLS 1.1, SSLv3, RC4, or static-RSA key exchange, and it does not support renegotiation. Legacy enterprise endpoints and some appliance/IoT stacks will fail to connect. There is no compatibility flag — the fix is fixing the peer, or using a provider/library that speaks the legacy protocol.

**Upgrade churn.** Because the project is pre-1.0, minor bumps break API. Notable inflection points: 0.22 introduced pluggable providers; 0.23 changed provider selection; 0.24 removed the in-crate providers entirely and forced explicit provider construction on `ClientConfig`/`ServerConfig`. Pin versions and budget migration time when bumping.

**FIPS.** aws-lc-rs offers a FIPS-validated mode; Rustls exposes it via a `fips` feature. This is a real differentiator versus most pure-Rust crypto, but it re-introduces C dependencies and validation-boundary constraints.

**Performance.** Maintainer benchmarks have shown Rustls matching or beating OpenSSL on handshake and bulk throughput[^5]; a `BENCHMARKING.md` and CI perf gates exist to catch regressions. Treat the published numbers as workload-specific and re-measure on your own hardware.

## When to Use / When Not

**Use when:**
- You want a memory-safe TLS stack with no `unsafe` in the protocol core.
- You value secure-by-default and are willing to drop legacy peers.
- You're in a Rust service or CLI and can use `tokio-rustls` / `hyper-rustls`.
- You need a deployable post-quantum handshake or a FIPS mode via aws-lc-rs.

**Avoid when:**
- You must interoperate with TLS 1.0/1.1, SSLv3, renegotiation, or other legacy behavior.
- You need a frozen, 1.0-stable API surface — Rustls still breaks across minor versions.
- Your platform can't build aws-lc-rs and you also need its exclusive features (PQ, some suites).
- You want batteries-included blocking sockets with zero glue — the sans-I/O core needs an integration layer.

## Alternatives

- sfackler/rust-openssl — mature, ubiquitous OpenSSL bindings; use when you need legacy protocol support or maximum interop, accepting C and its CVE history.
- sfackler/rust-native-tls — thin wrapper over the OS TLS stack (SChannel/SecureTransport/OpenSSL); use when you want to inherit the platform trust store and policy with minimal deps.
- aws/s2n-tls — AWS's C TLS library with a Rust binding; use when you want a small, FIPS-oriented C implementation backed by a vendor.
- openssl/openssl — the reference C implementation itself; use when a dependency or protocol feature is only available there.
- wolfSSL/wolfssl — compact C TLS for embedded/constrained targets; use when footprint and legacy-device support dominate.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2016-05 | Project started by @ctz[^1]. |
| 0.16 | 2019 | Early stabilization of the connection API. |
| 0.20 | 2021-09 | Major API rework; `ClientConfig`/`ServerConfig` builders. |
| 0.21 | 2023-03 | IP-address certs, further API cleanup. |
| 0.22 | 2023-12 | Pluggable `CryptoProvider`; aws-lc-rs and ring selectable[^4]. |
| 0.23 | 2024-03 | Provider-selection changes; post-quantum groundwork. |
| 0.24 | 2026 | Providers moved to separate crates; explicit provider now required[^2]. |

## References

[^1]: rustls/rustls repository and project membership. https://github.com/rustls/rustls
[^2]: rustls README — "Approach", "Cryptography providers", and licensing. https://github.com/rustls/rustls/blob/main/README.md
[^3]: ISRG Prossimo — Rustls initiative. https://www.memorysafety.org/initiative/rustls/
[^4]: `crypto::CryptoProvider` API documentation. https://docs.rs/rustls/latest/rustls/crypto/struct.CryptoProvider.html
[^5]: rustls benchmarking guide. https://github.com/rustls/rustls/blob/main/BENCHMARKING.md

## Tags

rust, tls, ssl, cryptography, network-security, memory-safety, tls13, sans-io, library, post-quantum
