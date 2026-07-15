# rust-openssl/rust-openssl

> OpenSSL bindings for Rust — a thin safe layer over the system's libssl/libcrypto, plus raw FFI.

[GitHub repo](https://github.com/sfackler/rust-openssl) ·
[Documentation](https://docs.rs/openssl) ·
[License: MIT OR Apache-2.0](https://github.com/rust-openssl/rust-openssl/blob/master/openssl/LICENSE-APACHE)

## Overview

rust-openssl is the canonical Rust binding to OpenSSL, maintained since 2013 by
Steven Fackler[^1]. It is split into two crates: `openssl`, the high-level safe
API (TLS `SslStream`, X.509 certificate handling, hashing, symmetric/asymmetric
crypto, PKCS helpers), and `openssl-sys`, the raw `unsafe extern "C"` FFI
declarations plus the build script that locates and links the native library.
The `openssl` crate is on the long-lived 0.10 line; `openssl-sys` on 0.9[^2].

Unlike pure-Rust TLS stacks, this crate implements no cryptography itself — it
delegates entirely to a C library at runtime, linking against whatever OpenSSL
(or LibreSSL, or BoringSSL) is present on the host and gating APIs behind `cfg`
flags detected at build time. That is its defining tension: you inherit
OpenSSL's exhaustively-exercised, FIPS-capable, hardware-accelerated
implementation along with its CVE stream, its platform-specific build friction,
and its version-fragmented API surface. The build script's job of finding a
compatible libssl is the single most common source of pain for new users.

It sits under a large fraction of the Rust networking ecosystem, most often via
`native-tls` (which selects OpenSSL on Linux), and transitively under HTTP
clients, database drivers, and messaging libraries that opt into a native-TLS
backend rather than rustls.

## Getting Started

```bash
cargo add openssl
```

On Debian/Ubuntu the build script needs the headers and `pkg-config`:

```bash
sudo apt-get install pkg-config libssl-dev
```

To sidestep system-library discovery entirely, build a copy of OpenSSL from
source and statically link it via the `vendored` feature[^3]:

```bash
cargo add openssl --features vendored
```

A minimal, self-contained example (SHA-256 via libcrypto):

```rust
use openssl::hash::{hash, MessageDigest};

fn main() -> Result<(), openssl::error::ErrorStack> {
    let digest = hash(MessageDigest::sha256(), b"hello world")?;
    for b in digest.iter() {
        print!("{:02x}", b);
    }
    println!();
    Ok(())
}
```

## Architecture / How It Works

The two-crate split is deliberate and mirrors the Rust `*-sys` convention:

- **`openssl-sys`** — declares the C symbols and owns the `build.rs`. At build
  time it locates libssl/libcrypto (via `pkg-config`, `vcpkg` on Windows, or the
  `OPENSSL_DIR`/`OPENSSL_LIB_DIR`/`OPENSSL_INCLUDE_DIR` environment variables),
  probes the header version, and emits `cargo:rustc-cfg` flags such as
  `ossl300`, `ossl110`, `libressl`, or `boringssl`. Downstream code is compiled
  against exactly the symbols the linked library actually exposes.
- **`openssl`** — wraps those raw pointers in owned types (`Ssl`, `SslContext`,
  `X509`, `PKey`, `BigNum`, …) with `Drop` impls that call the matching
  `*_free`. Foreign objects use the `foreign-types` crate to give each a
  reference (`&X509Ref`) and owned (`X509`) split.

Because the same source must compile against OpenSSL 1.0.2, 1.1.x, 3.x,
LibreSSL, and BoringSSL — libraries whose APIs differ in real ways — the crate
is dense with `#[cfg(...)]` gates; an API present only on OpenSSL 3 does not
exist when built against LibreSSL, so portable code targets the intersection.
Errors surface as `ErrorStack`, a direct wrapping of OpenSSL's thread-local
error queue, and TLS I/O is exposed as `SslStream<S>` over any `Read + Write`
transport, with `SslConnector`/`SslAcceptor` builders supplying Mozilla-derived
defaults rather than raw context configuration.

## Production Notes

**The build is the footgun, not the runtime.** "Could not find directory of
OpenSSL installation" is the archetypal failure. Resolutions, in order of
preference: install `libssl-dev` + `pkg-config` (Linux); on macOS point at the
Homebrew keg (`OPENSSL_DIR=$(brew --prefix openssl@3)`) since the system
LibreSSL is not linkable; or use `vendored` to compile OpenSSL from source and
statically link it, trading a longer, C-toolchain-dependent build for
reproducibility[^3].

**`vendored` pins you to a bundled OpenSSL version.** Static linking means a
security fix requires rebuilding and reshipping your binary rather than patching
a shared system library. For long-lived services that rely on the distro to push
libssl CVEs, dynamic linking is often the more defensible choice — decide per
deployment model, not by default.

**Cross-compilation is materially harder** than with rustls. You need an OpenSSL
built for the target architecture; the build script's host-oriented probing does
not transparently find target libraries, and `vendored` requires a C
cross-toolchain. Teams targeting many platforms often migrate to rustls
specifically to drop this dependency.

**Portability across backends is not free.** LibreSSL and BoringSSL are not
drop-in for every OpenSSL 3 symbol; features guarded by `cfg` can silently vanish
depending on what you link, so test against the exact library you ship. And
because the Rust layer is thin, a memory-safety issue in the linked C library is
still a memory-safety issue in your process — keep it current.

## When to Use / When Not

**Use when:**
- You need OpenSSL-specific capabilities: FIPS-validated modules, engines,
  hardware acceleration, or exotic X.509/PKCS/CMS operations rustls omits.
- You must interoperate with an existing OpenSSL-centric system or config.
- A dependency (e.g. `native-tls`, some DB drivers) already requires it.
- You want a mature, exhaustively-exercised crypto implementation and accept a C
  dependency.

**Avoid when:**
- You want a pure-Rust, no-C-toolchain build and easy cross-compilation — reach
  for rustls.
- You only need TLS client/server with modern defaults; rustls covers that with
  less build friction.
- Your deployment can't tolerate the system-library discovery dance and you
  don't want to vendor.

## Alternatives

- rustls/rustls — pure-Rust TLS, no OpenSSL/C dependency; use instead when you
  want painless builds and cross-compilation and don't need OpenSSL-specific
  features.
- briansmith/ring — low-level crypto primitives (not TLS); use when you need
  hashing/signing/AEAD without a full TLS stack.
- cloudflare/boring — bindings to BoringSSL forked from this crate; use when you
  specifically need BoringSSL semantics and Cloudflare's maintenance.
- aws/aws-lc-rs — ring-compatible API over AWS-LC; use when you want a
  FIPS-oriented, actively-funded crypto backend for rustls.
- sfackler/rust-native-tls — thin abstraction that selects OpenSSL, Secure
  Transport, or SChannel per OS; use when you want the platform's native stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-12 | Repo created; early bindings on the 0.7/0.8 lines[^1]. |
| openssl 0.10 / openssl-sys 0.9 | 2017 | Current long-lived API split; `foreign-types`-based ownership model[^2]. |
| — | 2018 | OpenSSL 1.1.x support alongside 1.0.2; opaque-struct API migration. |
| — | ~2021 | OpenSSL 3.0 support (`ossl300` cfg) added after 3.0's release. |
| openssl-sys 0.9.23 | 2026 | Latest tagged `-sys` release; 0.10/0.9 lines still current[^2]. |

## References

[^1]: rust-openssl repository, created 2013-12-28, maintained by Steven Fackler. https://github.com/sfackler/rust-openssl
[^2]: README, "Release Support" — supported release of `openssl` is 0.10 and `openssl-sys` is 0.9; new major versions at most once per year. https://github.com/rust-openssl/rust-openssl#release-support
[^3]: `openssl` crate features — `vendored` builds OpenSSL from source via the `openssl-src` crate and links it statically. https://docs.rs/openssl/latest/openssl/

## Tags

rust, openssl, tls, ffi, cryptography, bindings, ssl, x509, libcrypto, native-tls, systems
