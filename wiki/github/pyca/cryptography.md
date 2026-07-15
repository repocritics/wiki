# pyca/cryptography

> Python's de facto "cryptographic standard library" — OpenSSL primitives and a small set of safe recipes, wrapped for Python developers.

[GitHub repo](https://github.com/pyca/cryptography) ·
[Official website](https://cryptography.io) ·
[License: Apache-2.0 OR BSD-3-Clause](https://github.com/pyca/cryptography/blob/main/LICENSE)

## Overview

`cryptography` is maintained by the Python Cryptographic Authority (PyCA) and is
the package almost every serious Python project reaches for when it needs real
cryptography[^1]. First released in 2014, it deliberately splits its surface in
two: a small set of high-level "recipes" that are hard to misuse (Fernet), and a
large `hazmat` ("hazardous materials") layer that exposes raw primitives —
ciphers, hashes, KDFs, RSA/EC/Ed25519 keys, X.509, PKCS7/PKCS12 — for people who
know what they're doing. The naming is intentional: the library refuses to
pretend that low-level crypto is safe to use casually.

The defining tension is that `cryptography` is not itself a cryptographic
implementation — it is a binding. Symmetric ciphers, digests, and most asymmetric
math are executed by OpenSSL, which the project statically bundles into its
published wheels. Since version 3.4 (2021) the X.509, ASN.1, and an increasing
share of the higher-level logic have been rewritten in Rust, called through PyO3.
This gives strong memory-safety guarantees for the parsing code that
historically produces CVEs, at the cost of a build-time Rust toolchain
dependency that was, and occasionally still is, contentious[^2].

For the overwhelming majority of users this is invisible: `pip install
cryptography` fetches a prebuilt binary wheel with OpenSSL and the compiled Rust
already inside. The friction lands on anyone who must build from source — exotic
platforms, locked-down build farms, or distributions that predate the Rust
requirement.

## Getting Started

```bash
pip install cryptography
```

High-level authenticated symmetric encryption with Fernet (AES-128-CBC + HMAC):

```python
from cryptography.fernet import Fernet

key = Fernet.generate_key()          # store this somewhere safe
f = Fernet(key)
token = f.encrypt(b"A really secret message.")
assert f.decrypt(token) == b"A really secret message."
```

Dropping to the hazmat layer for AES-GCM (you own nonce management here):

```python
import os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

key = AESGCM.generate_key(bit_length=256)
nonce = os.urandom(12)               # NEVER reuse a nonce with the same key
ct = AESGCM(key).encrypt(nonce, b"secret", associated_data=b"header")
pt = AESGCM(key).decrypt(nonce, ct, associated_data=b"header")
```

## Architecture / How It Works

The package is three layers stacked on OpenSSL:

1. **Recipes** (`cryptography.fernet`) — opinionated, versioned, hard-to-misuse
   constructions. Fernet picks the algorithm, handles the IV, appends an HMAC,
   and timestamps the token. There are very few recipes on purpose.
2. **`hazmat.primitives`** — the pure-Python API surface: `ciphers`, `hashes`,
   `hmac`, `kdf`, `asymmetric` (RSA, DSA, EC, Ed25519/Ed448, X25519/X448, DH),
   `serialization` (PEM/DER/OpenSSH/PKCS8), and AEAD ciphers. These are thin
   objects that validate arguments and delegate.
3. **`hazmat.bindings` / Rust backend** — the actual calls. Historically this
   was a single "backend" abstraction over OpenSSL via cffi; the multi-backend
   design was removed and OpenSSL is now the only backend. The Rust layer
   (compiled as a native extension) owns ASN.1 parsing, X.509 path handling, and
   a growing set of primitives, calling OpenSSL through its own bindings.

**Why the Rust rewrite.** X.509 and ASN.1 parsing are the classic source of
memory-corruption CVEs across the entire TLS ecosystem. Moving that code from C
(or hand-written parsing) to memory-safe Rust removes an entire bug class at the
most dangerous layer. The tradeoff is build-chain weight, discussed below.

**Wheels bundle OpenSSL.** The manylinux/macOS/Windows wheels ship a pinned,
statically-linked OpenSSL. This means the OpenSSL version — and its CVE exposure
— is controlled by *your `cryptography` version*, not your operating system.
Upgrading `cryptography` is how you patch OpenSSL for this library; the system
`libssl` is irrelevant to the wheel path.

**ABI3.** Wheels are built against Python's stable ABI (abi3), so a single wheel
serves many CPython minor versions, which is why new Python releases usually work
immediately without waiting for a rebuild.

## Production Notes

**Building from source needs Rust + OpenSSL headers.** The common failure is on
a machine without a prebuilt wheel (old pip, uncommon architecture, Alpine/musl
before wheels existed, air-gapped build). You then need a Rust toolchain
(a recent stable rustc, tracked as a rising MSRV) plus OpenSSL development
headers, or the build fails hard. The fix is almost always "get the wheel"
(`pip install --upgrade pip` so pip recognizes the wheel tags) rather than
fighting the source build[^2].

**Nonce/IV discipline is on you in hazmat.** The AEAD APIs (`AESGCM`,
`ChaCha20Poly1305`) will happily let you reuse a nonce, which is catastrophic for
GCM (it can leak the authentication key). If you don't want to manage nonces,
use Fernet. This is the single most common way teams misuse the low-level layer.

**Fernet is not for large files or streaming.** `encrypt()` holds the entire
plaintext and ciphertext in memory and there is no streaming API. For big
payloads, chunk at the application layer or use a different construction.

**Deprecation cadence is brisk.** The project drops end-of-life Python versions
and old OpenSSL versions on a steady schedule, and emits `CryptographyDeprecationWarning`
ahead of removals. Pinning `cryptography` and reading the changelog before major
bumps is worth it: APIs like the old `backend=` argument and various legacy
serialization helpers have been removed over time.

**Version coupling with pyOpenSSL / others.** Libraries such as pyOpenSSL,
requests (via urllib3), and paramiko depend on `cryptography`. A `cryptography`
major bump occasionally breaks a lagging dependent until it widens its pin;
resolve conflicts by upgrading the dependent, not by holding `cryptography` back
on an unpatched OpenSSL.

**FIPS.** The bundled OpenSSL is not a FIPS-validated module. Environments with
FIPS mandates need a system OpenSSL FIPS provider and a source build linked
against it — this is explicitly outside the happy path.

## When to Use / When Not

**Use when:**
- You need correct, maintained cryptography in Python and want the ecosystem-standard choice.
- You want safe defaults (Fernet) with an escape hatch to primitives (hazmat) in one package.
- You need X.509 certificate parsing/generation, PKCS12, or key serialization.
- You want OpenSSL's algorithm coverage and performance without writing the bindings yourself.

**Avoid when:**
- You cannot provide prebuilt wheels *and* cannot install a Rust toolchain in your build environment.
- You only need password hashing (use a dedicated bcrypt/argon2 library) or simple hashing/HMAC (use the stdlib).
- You require a FIPS-validated module out of the box.
- You want a pure-Python, dependency-free implementation for a constrained target.

## Alternatives

- Legrandin/pycryptodome — self-contained (no OpenSSL/Rust build dep); use when you can't install a Rust toolchain or want minimal system dependencies.
- pyca/pynacl — libsodium bindings; use when you want opinionated modern NaCl primitives (box/secretbox) with fewer choices to get wrong.
- tink-crypto/tink-py — Google's misuse-resistant API with key management; use when you want envelope encryption and hard-to-misuse key handling over raw primitives.
- pyca/bcrypt — use when your only need is password hashing, not general crypto.
- python/cpython (stdlib `hashlib`, `hmac`, `secrets`) — use for hashing, HMAC, and secure token generation with zero third-party dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.2 | 2014-05 | Early public releases; Fernet and initial hazmat primitives[^1]. |
| 1.0 | 2015-08 | First stable API. |
| 2.0 | 2017-07 | Expanded asymmetric + serialization support. |
| 3.0 | 2020-07 | Dropped Python 2.7; OpenSSL/API cleanups. |
| 3.4 | 2021-02 | Introduced the Rust build dependency; source-build friction begins[^2]. |
| 35.0.0 | 2021-09 | Switched to a new major-version scheme[^3]. |
| 42.0.0 | 2024-01 | Continued OpenSSL 3.x support and API deprecations[^3]. |
| 45.0.0 | 2025 | Ongoing OpenSSL bundling + primitive migration to Rust[^3]. |

## References

[^1]: pyca/cryptography documentation and project overview. https://cryptography.io/
[^2]: Discussion of the Rust build-time dependency introduced in cryptography 3.4 (build requirements, affected platforms). https://cryptography.io/en/latest/installation/
[^3]: cryptography changelog / release notes. https://cryptography.io/en/latest/changelog/

## Tags

python, cryptography, openssl, rust-bindings, x509, tls, security, encryption, pyca, hazmat, aead
