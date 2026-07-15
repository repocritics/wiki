# openssl/openssl

> General-purpose TLS and cryptography library — the C implementation that most of the internet's encryption ultimately depends on.

[GitHub repo](https://github.com/openssl/openssl) ·
[Official website](https://openssl-library.org/) ·
[License: Apache-2.0](https://github.com/openssl/openssl/blob/master/LICENSE.txt)

## Overview

OpenSSL is a cryptography and transport-security toolkit written in C. It provides `libcrypto` (primitives: ciphers, digests, public-key, X.509, ASN.1, random), `libssl` (TLS 1.0–1.3, DTLS, and QUIC on top of `libcrypto`), and the `openssl` command-line tool. It descends from Eric A. Young and Tim J. Hudson's SSLeay, and the OpenSSL project has maintained it since 1998[^1]. Practically every language runtime, Linux distribution, load balancer, and networked daemon links against it directly or ships a fork of it, which makes OpenSSL less a "library you choose" than "the crypto substrate you inherit".

The defining tension is that OpenSSL is simultaneously the most ubiquitous and the most criticized crypto library in existence. Its API surface is enormous, historically under-documented, and full of low-level functions that are easy to misuse; its codebase carries decades of accreted ABI compatibility. The 2014 Heartbleed vulnerability (CVE-2014-0160)[^2] exposed how thinly resourced the project was, triggered the LibreSSL and BoringSSL forks, and led to the Core Infrastructure Initiative funding that professionalized maintenance.

The 3.0 release (2021) was a ground-up architectural reset: a new **provider** model for algorithm implementations, a FIPS 140-validated module, and a relicense from the GPL-incompatible dual OpenSSL/SSLeay license to Apache-2.0[^3]. That reset is still working its way through the ecosystem — 3.0 shipped real performance regressions and API churn that many downstreams are only now past.

## Getting Started

Most users should link the copy their OS vendor ships rather than build from source. To build from source:

```bash
git clone https://github.com/openssl/openssl.git
cd openssl
./Configure          # detects platform; add options e.g. enable-fips, --prefix=/opt/ssl
make -j"$(nproc)"
make test
sudo make install
```

Use the CLI for common tasks:

```bash
# self-signed cert + key
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# inspect a server's TLS handshake and certificate chain
openssl s_client -connect example.com:443 -servername example.com </dev/null

# hash and encrypt
openssl dgst -sha256 file.bin
```

The recommended C interface is the high-level **EVP** API, not the low-level primitive calls:

```c
#include <openssl/evp.h>

EVP_MD_CTX *ctx = EVP_MD_CTX_new();
EVP_DigestInit_ex(ctx, EVP_sha256(), NULL);
EVP_DigestUpdate(ctx, data, len);
unsigned char out[EVP_MAX_MD_SIZE];
unsigned int outlen;
EVP_DigestFinal_ex(ctx, out, &outlen);
EVP_MD_CTX_free(ctx);
```

## Architecture / How It Works

The library is layered. `libcrypto` sits at the bottom; `libssl` is a consumer of it. Above the raw algorithm implementations sits the **EVP** abstraction, which is the interface application code is meant to use — it dispatches to whatever concrete implementation is currently active for a named algorithm.

Since 3.0 that dispatch goes through **providers**[^3]. A provider is a module that exposes algorithm implementations; OpenSSL ships several:

- **default** — the normal implementations, loaded automatically.
- **fips** — the FIPS 140-validated cryptographic module, loaded when the application opts in.
- **legacy** — deprecated/insecure algorithms (MD2, RC4, DES, Blowfish, etc.) moved out of the default set.
- **base** — encoders/decoders and serialization that are not cryptographic per se.

At runtime, an EVP operation resolves an algorithm by *fetching* it: OpenSSL matches the requested name against a **property query string** (e.g. `provider=fips` or `fips=yes`) across loaded providers. Providers replaced the older **ENGINE** interface, which is now deprecated. This model is what enables the FIPS module to be a drop-in swap without recompiling applications, but it also inserts a lookup on the operation path (see Production Notes).

`libssl` implements the TLS/DTLS state machine and record layer on top of `libcrypto`. QUIC support (client-side from 3.2, with server-side arriving in the 3.5 line) is integrated here rather than delegated to a separate library, and includes OpenSSL's own event/timeout handling for the QUIC transport.

Configuration is driven by the Perl-based `Configure` script (there is no autotools/CMake), which generates platform-specific makefiles from a large set of `enable-*`/`no-*` options and per-target `Configurations/*.conf` definitions. Runtime behavior (providers to load, default properties, algorithm policy) is controlled through `openssl.cnf`.

## Production Notes

**The 3.0 fetch performance regression.** The single most-cited operational problem with the 3.x series. Because EVP operations resolve implementations via provider fetch, code that repeatedly does one-shot operations (e.g. `EVP_EncryptInit_ex` with a `EVP_get_cipherbyname`/implicit-fetch pattern in a hot loop) pays a lock-contended lookup each time. High-connection-rate servers saw measurable throughput drops moving 1.1.1 → 3.0[^4]. The mitigation is to **explicitly fetch once and reuse**: call `EVP_MD_fetch` / `EVP_CIPHER_fetch` at startup, cache the returned object, and pass it to init. Later 3.x releases reduced but did not eliminate the overhead.

**Version and support policy.** OpenSSL designates specific releases as LTS (1.0.2, 1.1.1, 3.0, and 3.5) and supports them longer; non-LTS feature releases have shorter windows. Pin to an LTS line for anything you will not revisit frequently, and track the project's release-strategy page — EOL dates are hard cutoffs after which no security fixes ship. 1.1.1 (which introduced TLS 1.3) reached EOL in September 2023, stranding a large amount of software on unsupported crypto.

**ABI and SONAME.** The 1.1.x → 3.x transition changed the SONAME and removed/deprecated large swaths of low-level API (direct `RSA`/`EC_KEY`/`EVP_PKEY` struct access, the low-level cipher calls). Porting a nontrivial C codebase across it is real work, not a recompile: opaque structs mean many `struct->field` accesses become getter/setter calls, and engines must be reworked as providers.

**Two libraries on one system.** Because so many things vendor or link OpenSSL, hosts routinely have multiple major versions present (system 3.x plus an app-bundled copy). Symbol clashes and "loaded the wrong `openssl.cnf`" surprises are common; check `openssl version -a` and `OPENSSL_CONF`/`OPENSSL_MODULES` when behavior differs from expectation.

**FIPS.** The FIPS provider must be installed and self-tested (`fipsinstall`) and referenced from config; simply building `enable-fips` is not sufficient to be in a validated state, and validation applies to a specific module version, not to OpenSSL generally.

**CVE cadence.** OpenSSL ships coordinated security advisories on a schedule with severity ratings. Notably, CVE-2022-3602/3786 (the punycode buffer overflows in 3.0.7) were pre-announced as "Critical" then downgraded to "High" on release[^5] — a reminder to read the actual advisory rather than react to the severity label alone.

## When to Use / When Not

**Use when:**
- You need broad protocol/algorithm coverage (TLS 1.3, DTLS, QUIC, S/MIME, X.509, a very wide cipher set) in one library.
- You require FIPS 140 validation — OpenSSL's validated module is the most widely accepted option.
- You are integrating with an existing C/C++ stack that already assumes the OpenSSL API, or need maximum platform portability.

**Avoid when:**
- You control both endpoints and want a smaller, opinionated, memory-safe stack — a modern TLS library with a narrower API will be easier to use correctly.
- You are writing new networking code in Rust/Go and can use a native library instead of FFI into a large C surface.
- You want a minimal footprint for embedded/constrained targets — OpenSSL is comparatively large.

## Alternatives

- google/boringssl — Google's fork, tuned for Chromium/Android; no API/ABI stability guarantees, so use it only when you can track it closely.
- libressl-portable/portable (LibreSSL) — OpenBSD's post-Heartbleed fork; cleaner code, drop-in-ish for 1.0.x-era APIs, lags on newer features.
- Mbed-TLS/mbedtls — small, readable C library for embedded and constrained systems; use when footprint and auditability beat breadth.
- wolfSSL/wolfssl — commercial-friendly embedded TLS with FIPS options; use for MCUs and real-time targets.
- rustls/rustls — memory-safe TLS in Rust; use for new Rust services that don't need OpenSSL's algorithm breadth or FIPS module.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9.1c | 1998-12 | First release under the OpenSSL name, forked from SSLeay[^1]. |
| 1.0.1 | 2012-03 | TLS 1.1/1.2; the branch later found vulnerable to Heartbleed. |
| — | 2014-04 | Heartbleed (CVE-2014-0160) disclosed; LibreSSL/BoringSSL forks follow[^2]. |
| 1.0.2 | 2015-01 | LTS line (EOL 2019-12-31). |
| 1.1.1 | 2018-09 | TLS 1.3; LTS line (EOL 2023-09-11). |
| 3.0 | 2021-09 | Providers, FIPS module, Apache-2.0 relicense; LTS[^3]. |
| 3.2 | 2023-11 | Client-side QUIC. |
| 3.5 | 2025-04 | LTS line; server-side QUIC support. |
| 4.0 | 2026 | Current major release line (see docs.openssl.org/4.0). |

## References

[^1]: OpenSSL project history and SSLeay lineage. https://www.openssl.org/
[^2]: The Heartbleed Bug (CVE-2014-0160). https://heartbleed.com/
[^3]: OpenSSL 3.0 migration guide (providers, FIPS, license change). https://docs.openssl.org/master/man7/ossl-guide-migration/
[^4]: OpenSSL 3.0 performance / fetch overhead discussion. https://github.com/openssl/openssl/issues/20286
[^5]: OpenSSL Security Advisory, CVE-2022-3602 / CVE-2022-3786 (1 November 2022). https://www.openssl.org/news/secadv/20221101.txt

## Tags

c, cryptography, tls, ssl, security, openssl, x509, quic, fips, network-security, library
