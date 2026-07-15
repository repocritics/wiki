# Mbed-TLS/mbedtls

> A small-footprint C library implementing TLS/DTLS, X.509, and the PSA Cryptography API — built for embedded and constrained systems.

[GitHub repo](https://github.com/Mbed-TLS/mbedtls) ·
[Official website](https://www.trustedfirmware.org/projects/mbed-tls/) ·
[License: Apache-2.0 OR GPL-2.0-or-later](https://github.com/Mbed-TLS/mbedtls/blob/development/LICENSE)

## Overview

Mbed TLS is a portable C library implementing the TLS and DTLS protocols, X.509 certificate handling, and a broad set of cryptographic primitives. It began life as PolarSSL, an independent project focused on a clean, auditable, dependency-light TLS stack for embedded targets. Arm acquired the project (via Offspark) in 2014 and rebranded it "mbed TLS"; in 2020 stewardship moved to the vendor-neutral TrustedFirmware.org / Linaro umbrella, which is where it is governed today[^1]. The library's defining trait is code footprint: it is written in portable C99, has a highly granular compile-time configuration, and can be trimmed to fit microcontrollers with tens of kilobytes of flash.

The central architectural story of the modern library is the migration to the **PSA Cryptography API**[^2] — Arm's Platform Security Architecture crypto interface, designed around opaque key handles and pluggable secure-element / hardware backends rather than raw key material in application memory. Since the 4.0 release the cryptographic core has been split out into a separate repository, **TF-PSA-Crypto**, which `mbedtls` consumes as a submodule; the top-level `mbedtls` repo now provides the TLS and X.509 layers on top of it[^3]. This split is the biggest structural change in the project's history and the source of most of the friction in upgrading from the 2.x/3.x era.

The tradeoff Mbed TLS makes is breadth-for-footprint. It does not chase every TLS extension or the absolute performance ceiling that OpenSSL or BoringSSL target on servers; instead it optimizes for being small, readable, constant-time where it matters, and easy to port to a new RTOS or bare-metal environment. That makes it dominant in firmware and IoT and a poor fit as a drop-in for a high-throughput server terminating millions of handshakes.

## Getting Started

Mbed TLS builds with CMake and a C99 toolchain. Three libraries are produced: `libtfpsacrypto` (crypto), `libmbedx509` (certificates), and `libmbedtls` (TLS), with a strict dependency order `mbedtls → mbedx509 → tfpsacrypto`[^3].

```bash
git clone https://github.com/Mbed-TLS/mbedtls.git
cd mbedtls
git submodule update --init --recursive   # pulls framework + tf-psa-crypto
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
ctest --test-dir build                    # requires Python to generate test files
```

A minimal PSA Crypto program — hashing with SHA-256 through the opaque-handle API:

```c
#include "psa/crypto.h"
#include <stdio.h>

int main(void) {
    if (psa_crypto_init() != PSA_SUCCESS) return 1;

    const uint8_t msg[] = "hello";
    uint8_t hash[32];
    size_t out_len;

    psa_status_t s = psa_hash_compute(PSA_ALG_SHA_256,
                                      msg, sizeof(msg) - 1,
                                      hash, sizeof(hash), &out_len);
    if (s != PSA_SUCCESS) return 1;

    for (size_t i = 0; i < out_len; i++) printf("%02x", hash[i]);
    printf("\n");
    return 0;
}
```

Configuration is compile-time: `include/mbedtls/mbedtls_config.h` controls X.509/TLS features, and `tf-psa-crypto/include/psa/crypto_config.h` controls crypto and platform options. Feature toggles are `#define` macros; `scripts/config.py` edits them programmatically.

## Architecture / How It Works

Mbed TLS is organized as three layered libraries plus a submodule-provided crypto core:

- **TF-PSA-Crypto** (submodule since 4.0) — the cryptographic primitives (ciphers, hashes, MACs, ECC, RSA, DRBGs) exposed through the PSA Cryptography API. Also shipped under its legacy name `libmbedcrypto` for backward compatibility. Keys are referenced by `psa_key_id_t` handles, and the PSA driver interface lets a hardware crypto accelerator or secure element back a key without the application ever touching raw bytes.
- **libmbedx509** — X.509 certificate, CRL, and CSR parsing / verification. Depends only on the crypto core.
- **libmbedtls** — the TLS 1.2 / 1.3 and DTLS state machines, session handling, and record layer, built on top of the two below it.

Everything is driven by **compile-time configuration**. Unlike OpenSSL, which is largely runtime-configured, Mbed TLS expects you to `#define`/`#undef` features in the config header so the linker can dead-strip anything you don't use. This is what enables the small footprint but means "which features are available" is a property of your build, not a runtime query.

Portability is deliberate: the library targets portable C99 with a short list of documented platform assumptions (8-bit bytes, two's-complement integers, all-zero null pointers, no mixed-endian platforms, `int`/`size_t` ≥ 32 bits)[^4]. Platform abstraction macros (`MBEDTLS_PLATFORM_*`) let you redirect `malloc`, `printf`, entropy sources, and time onto an RTOS or bare-metal environment.

The development branch does not check in generated source and test files; they are produced by Python/Perl scripts at configure time (or via `make_generated_files.py`) and only bundled into official release tarballs. This trips up people who clone `development` expecting to build without Python installed.

## Production Notes

**The 3.x → 4.0 migration is the major upgrade hazard.** The crypto core moving to TF-PSA-Crypto, the PSA API becoming the primary surface, and the removal of long-deprecated legacy crypto entry points mean this is not a drop-in bump. Teams pinned to the legacy `mbedtls_*` crypto calls should budget real porting time or stay on the **3.6 LTS** line, which is the recommended long-term-stable target and receives security fixes on a defined support window[^5].

**Submodules are load-bearing.** A plain `git clone` without `--recurse-submodules` (or a follow-up `git submodule update --init --recursive`) yields a tree that will not build. The 3.6 LTS branch carries only the `framework` submodule; `development` and 4.x carry both `framework` and `tf-psa-crypto`. CI that caches checkouts frequently breaks on this.

**Compile-time config is a footgun and a feature.** Because features are stripped at build time, a config mismatch between the library build and the consuming application (different `mbedtls_config.h`) produces subtle ABI and behavioral bugs rather than clean errors. Keep the config header under version control alongside the application, and rebuild the library and app against the same config.

**Performance is footprint-first, not throughput-first.** On constrained MCUs Mbed TLS is appropriate and well-tuned, and hardware acceleration via PSA drivers can offload ECC/AES to a crypto peripheral. On a general-purpose server it will not match OpenSSL/BoringSSL throughput; it is not the right tool for terminating TLS at high connection rates.

**Security process.** Vulnerabilities are reported privately to the TrustedFirmware security list, and advisories are published with each affected release[^6]. Because the library is embedded in firmware that is expensive to update in the field, track the security announce channel and the LTS branch rather than assuming devices auto-update.

**Test suite needs Python and Perl.** Unit tests are generated from `.function` + `.data` files by Python; interop tests (`ssl-opt.sh`, `compat.sh`) need Perl plus OpenSSL/GnuTLS installed. The project provides Docker images mirroring its CI if you don't want to install the toolchain matrix by hand.

## When to Use / When Not

**Use when:**
- You're building firmware / IoT / embedded systems where flash and RAM are constrained and code auditability matters.
- You want a hardware-backed key story via the PSA driver interface (secure element, TPM-like crypto peripheral).
- You need a readable, portable C TLS stack you can port to a custom RTOS or bare-metal target.
- You want a permissively licensed (Apache-2.0) alternative to copyleft or legacy-licensed stacks.

**Avoid when:**
- You're terminating TLS at high volume on servers — OpenSSL or BoringSSL are the throughput-oriented choices.
- You need a large surface of TLS extensions, protocols, or a mature CLI toolchain (`openssl` command) out of the box.
- Your team can't accommodate compile-time configuration discipline across library and application builds.
- You need FIPS 140 validated binaries off the shelf (that pathway is more established around OpenSSL / wolfSSL).

## Alternatives

- openssl/openssl — the ubiquitous general-purpose TLS/crypto stack; use it when you need maximum protocol/extension coverage, server throughput, and the `openssl` CLI, and can absorb its size and complexity.
- wolfSSL/wolfssl — another embedded-focused TLS library; use it when you need commercial support, FIPS validation, or DO-178 / hardware-cert pathways for regulated firmware.
- google/boringssl — Google's OpenSSL fork; use it when you're inside the Chromium/Android ecosystem, but note it offers no API/ABI stability guarantees for external consumers.
- aws/s2n-tls — Amazon's minimal TLS implementation; use it when you want a small, security-audited server-side TLS library on Linux rather than an embedded/MCU target.
- libressl — OpenBSD's OpenSSL fork; use it when you want a de-crufted, security-focused OpenSSL API-compatible drop-in on POSIX systems.

## History

| Version | Date | Notes |
|---------|------|-------|
| PolarSSL 0.x | 2006–2009 | Original independent project; clean-room embedded TLS. |
| — | 2014 | Arm acquires Offspark; project rebranded from PolarSSL to mbed TLS. |
| 2.0 | 2015 | First release under the mbed TLS name; API reorganization. |
| — | 2020 | Governance moved to TrustedFirmware.org (Linaro), vendor-neutral. |
| 3.0 | 2021-07 | Major cleanup; removal of long-deprecated APIs; PSA Crypto promoted. |
| 3.6 LTS | 2024 | Long-term-support branch; TLS 1.3 mature; recommended stable target[^5]. |
| 4.0 | 2026 | Crypto core split into TF-PSA-Crypto submodule; PSA API as primary surface[^3]. |

## References

[^1]: TrustedFirmware.org — Mbed TLS project page. https://www.trustedfirmware.org/projects/mbed-tls/
[^2]: PSA Certified Crypto API specification (Arm). https://arm-software.github.io/psa-api/
[^3]: Mbed TLS README — library structure, TF-PSA-Crypto submodule, build order. https://github.com/Mbed-TLS/mbedtls/blob/development/README.md
[^4]: Mbed TLS README — "Porting Mbed TLS", platform requirements. https://github.com/Mbed-TLS/mbedtls/blob/development/README.md
[^5]: Mbed TLS branch policy and LTS support windows. https://github.com/Mbed-TLS/mbedtls/blob/development/BRANCHES.md
[^6]: Mbed TLS security policy. https://github.com/Mbed-TLS/mbedtls/blob/development/SECURITY.md

## Tags

c, tls, dtls, cryptography, x509, embedded, psa-crypto, security, iot, ssl, trustedfirmware
