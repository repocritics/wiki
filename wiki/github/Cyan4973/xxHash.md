# Cyan4973/xxHash

> A non-cryptographic hash family (XXH32/XXH64/XXH3) that runs at RAM-bandwidth speed and is the de facto checksum for LZ4, Zstandard, and much of the compression world.

[GitHub repo](https://github.com/Cyan4973/xxHash) ·
[Official website](http://www.xxhash.com/) ·
License: BSD-2-Clause (library) + GPL-2.0 (xxhsum CLI)

## Overview

xxHash is a small C library of non-cryptographic hash functions written by Yann Collet (Cyan4973), the same author as LZ4 and Zstandard. Its stated design goal is throughput: hashing "at RAM speed limits", meaning the algorithm is fast enough that main-memory bandwidth, not CPU, is usually the bottleneck on large inputs. It is the checksum and dictionary hash inside LZ4 and Zstd, and by extension ships inside the Linux kernel, and countless databases, filesystems, and build tools[^1].

The library exposes three generations. XXH32 (2012) and XXH64 (2014) are simple, portable, scalar hashes. XXH3 and its 128-bit sibling XXH128, stabilized in v0.8.0 (2020), are a substantial redesign that uses SIMD (SSE2/AVX2/AVX512/NEON/VSX) and is markedly faster on both large and small inputs[^2]. XXH3 is now the recommended default; XXH32/XXH64 are kept for backward compatibility and platforms without 64-bit arithmetic.

The defining tension is right in the name: this is not a cryptographic hash. It passes the SMHasher quality suite for dispersion, avalanche, and collision behavior[^3], but it is not collision-resistant against an adversary and must never be used where an attacker controls inputs and can profit from forced collisions (password hashing, MACs, deduplication of untrusted data, HashDoS-exposed hash tables). Within its lane — integrity checks, hash tables, bloom filters, content addressing of trusted data — it is one of the fastest options available.

## Getting Started

The library is two files, `xxhash.h` and `xxhash.c`, and is trivially vendorable. Package managers also carry it (`apt install libxxhash-dev`, `brew install xxhash`, vcpkg `xxhash`, Conan, etc.).

```c
#include "xxhash.h"

/* one-shot 64-bit hash of a single buffer */
XXH64_hash_t h64 = XXH64(buffer, size, /*seed=*/0);

/* XXH3 is the modern default — faster, better small-key behavior */
XXH64_hash_t h3  = XXH3_64bits(buffer, size);
XXH128_hash_t h128 = XXH3_128bits(buffer, size);
```

For incremental data, use the streaming API: `XXH3_createState()` → `XXH3_64bits_reset()` → `XXH3_64bits_update()` (any number of times) → `XXH3_64bits_digest()` → `XXH3_freeState()`. A common trick is header-only mode: define `XXH_INLINE_ALL` before including `xxhash.h` to inline everything, which is a large win for small, compile-time-constant key lengths.

The repository also builds `xxhsum`, a CLI mirroring `md5sum`/`sha1sum` semantics (`xxhsum -c` to verify a checksum file).

## Architecture / How It Works

The core is single-header-friendly C89-compatible code with heavy use of macros for portability across compilers, endianness, and instruction sets. Output is identical on little- and big-endian platforms by construction.

XXH3 is where the engineering lives. It is a vectorized accumulator design: input is processed in blocks against a fixed 192-byte secret, with per-lane multiply-and-mix operations that map onto SIMD registers. The build auto-selects a vector backend (`XXH_VECTOR` = SCALAR/SSE2/AVX2/AVX512/NEON/VSX) at compile time, and can optionally resolve it at *runtime* via the dispatch layer (`xxh_x86dispatch.c`, `DISPATCH=1`) so a single binary picks AVX2/AVX512 based on the host CPU. Small inputs take dedicated short-length code paths so that fixed initialization/finalization cost does not dominate — this is the main reason XXH3 beats XXH64 on keys of a few bytes.

Behavior is controlled almost entirely through compile-time macros rather than runtime options. The important ones: `XXH_INLINE_ALL` / `XXH_PRIVATE_API` (header-only inlining), `XXH_NAMESPACE` (symbol prefixing to avoid collisions when multiple copies are vendored), `XXH_STATIC_LINKING_ONLY` (exposes struct internals for static allocation, at the cost of ABI stability), `XXH_FORCE_MEMORY_ACCESS` (how unaligned reads are done), and a family of `XXH_NO_*` flags (`XXH_NO_XXH3`, `XXH_NO_LONG_LONG`, `XXH_NO_STREAM`, `XXH_NO_STDLIB`) for trimming binary size and dependencies in embedded targets.

The library has effectively no runtime dependencies; with the `XXH_NO_STDLIB` / `XXH_memcpy` redirection macros it can drop `<stdlib.h>` and `<string.h>` entirely, which is why it lands in freestanding and kernel environments.

## Production Notes

- **Not stable across algorithms, stable within one.** XXH32, XXH64, and XXH3 produce different values; there is no interoperability between them. XXH3's output was *not* frozen until v0.8.0 — code that persisted XXH3 hashes from a v0.7.x preview will not match v0.8.0+. Never store XXH3 values computed by pre-0.8.0 builds and expect them to reproduce.
- **Seeds and secrets.** XXH3 supports both a 64-bit seed and a custom secret. If you persist hashes, pin the seed/secret; changing it silently changes every hash.
- **`XXH_STATIC_LINKING_ONLY` breaks ABI.** It exposes state struct layouts for static allocation, which means the struct size can change between versions. Fine for static linking within one build; a trap for anything that dynamically links `libxxhash`.
- **Auto-vectorization is usually a pessimization for XXH32/XXH64.** The source deliberately suppresses it; enabling `XXH_ENABLE_AUTOVECTORIZE` can make those two *slower*. XXH3 is the one designed for SIMD.
- **RAM-bound, not cache-bound claims.** The headline "faster than RAM" numbers only materialize when data is already in L3 or better; on cold large buffers you hit the memory wall regardless of the hash.
- **Security boundary is the real footgun.** Because it is fast and ubiquitous, xxHash gets misused as a security primitive. It offers zero preimage/collision resistance against an adversary. Anything touching untrusted or attacker-influenced input needs SipHash (HashDoS-resistant table keying) or a real cryptographic hash (BLAKE3/SHA-2).
- **Namespacing when vendoring.** Two libraries that both statically vendor xxHash into one process cause symbol clashes; use `XXH_NAMESPACE` to prefix.

## When to Use / When Not

**Use when:**
- You need maximum-throughput integrity checks or content hashing of *trusted* data.
- You are keying an in-memory hash table or bloom filter and inputs are not attacker-controlled.
- You want a tiny, dependency-free, portable C hash that compiles anywhere, including embedded/freestanding.
- You need to match LZ4/Zstd internal checksums, or interoperate with the many language bindings.

**Avoid when:**
- Inputs are adversarial and a forced collision benefits the attacker (use SipHash, or a cryptographic hash).
- You need cryptographic guarantees — signatures, MACs, password storage, dedup of untrusted uploads.
- You need to persist hashes across a migration from XXH64 to XXH3, or from pre-0.8.0 XXH3 (values differ).

## Alternatives

- google/highwayhash — SIMD hash with a keyed, HashDoS-resistant variant; use when you need speed *and* resistance to adversarial collisions.
- veorq/SipHash — slower but designed for hash-table keying against untrusted input; use for any HashDoS-exposed map.
- BLAKE3-team/BLAKE3 — use when you actually need a cryptographic hash but still want high throughput.
- rurban/smhasher — not a hash but the test harness; use to independently evaluate any of these on quality/speed.
- wangyi-fudan/wyhash — comparable non-cryptographic speed, often faster on tiny keys; use when small-key throughput dominates and you can accept a smaller ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| XXH32 | 2012 | Original 32-bit algorithm, spun out of LZ4's checksum needs[^1]. |
| XXH64 | 2014-08 | 64-bit variant (first version contributed by @JCash)[^4]. |
| v0.7.x | 2019 | XXH3 introduced as experimental/preview — output not yet frozen. |
| v0.8.0 | 2020-08 | XXH3 and XXH128 stabilized; output format frozen[^2]. |
| v0.8.1–0.8.2 | 2022–2024 | Portability, build-system, and packaging fixes; API stable. |

## References

[^1]: xxHash project site and algorithm overview. http://www.xxhash.com/
[^2]: "xxHash v0.8.0 — XXH3 is now stable." Release notes. https://github.com/Cyan4973/xxHash/releases/tag/v0.8.0
[^3]: SMHasher hash-quality test suite (Appleby; extended fork by rurban). https://github.com/rurban/smhasher
[^4]: xxHash README, "Special Thanks" — @JCash contributed the first XXH64. https://github.com/Cyan4973/xxHash

## Tags

c, hashing, non-cryptographic-hash, checksum, xxh3, simd, performance, hash-table, data-integrity, embedded, algorithm
