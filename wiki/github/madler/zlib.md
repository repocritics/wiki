# madler/zlib

> The DEFLATE compression library that ships inside almost everything — PNG, git, the Linux kernel, ssh, HTTP gzip, and most language runtimes.

[GitHub repo](https://github.com/madler/zlib) ·
[Official website](https://zlib.net/) ·
License: Zlib (permissive, BSD-like)[^1]

## Overview

zlib is a C library for lossless data compression built around the DEFLATE algorithm — LZ77 dictionary matching followed by Huffman coding — as standardized in RFC 1951[^2]. It also implements the two container formats layered on top of raw DEFLATE: the zlib format (RFC 1950, a 2-byte header plus an Adler-32 checksum) and the gzip format (RFC 1952, the format of `.gz` files, with a CRC-32). Jean-loup Gailly wrote the compression side and Mark Adler wrote the decompression side; the same two authors also wrote the original `gzip` utility, and the entire library is by them with no third-party code[^3].

It is one of the most widely deployed pieces of software in existence. libpng depends on it, git uses it for object storage and the pack protocol, the Linux kernel bundles a copy, OpenSSH, PostgreSQL, and countless others link it, and virtually every language standard library (Python's `zlib`/`gzip`, Java's `java.util.zip`, Node's `zlib`, Go's `compress/flate`) either wraps it or reimplements its formats. Because of this, zlib's on-disk and on-wire formats are effectively frozen infrastructure — the compressed-data compatibility guarantee is the whole point.

The defining tension is age versus ubiquity. The core is 30-year-old C with a hand-tuned but scalar (non-SIMD) inner loop, developed at a deliberately glacial pace by a maintainer who prioritizes correctness and format stability over throughput. That conservatism is exactly why it can be trusted as a dependency by half the software industry — and exactly why performance-sensitive users increasingly reach for optimized forks instead.

## Getting Started

Most systems already have it (`-lz`); to build from source:

```bash
./configure
make test      # builds and runs test/example.c + minigzip
make install
```

Minimal one-shot round-trip using the whole-buffer helpers:

```c
#include <zlib.h>
#include <string.h>
#include <stdio.h>

int main(void) {
    const char *msg = "hello hello hello hello";
    unsigned char comp[64], out[64];
    uLongf clen = sizeof(comp), olen = sizeof(out);

    compress(comp, &clen, (const Bytef *)msg, strlen(msg) + 1);
    uncompress(out, &olen, comp, clen);

    printf("%lu -> %lu -> %s\n", strlen(msg) + 1, clen, out);
    return 0;
}
// cc example.c -lz
```

`compress()`/`uncompress()` are convenience wrappers; real workloads use the streaming API (`deflateInit`/`deflate`/`deflateEnd` and the `inflate*` counterparts) so data need not fit in memory at once.

## Architecture / How It Works

The public surface is a single header, `zlib.h`, which is also the canonical documentation — there are no man pages, and reading `zlib.h` is the intended way to learn the API[^3]. There are three layers of interface:

1. **Streaming (`z_stream`)** — the core. You feed `next_in`/`avail_in`, drain `next_out`/`avail_out`, and call `deflate()`/`inflate()` in a loop until `Z_STREAM_END`. All state lives in a caller-visible `z_stream` struct with pluggable `zalloc`/`zfree` allocators. This is what everything else is built on.
2. **One-shot (`compress`/`uncompress`)** — buffer-to-buffer wrappers for when the data already fits in memory.
3. **gzip file I/O (`gzopen`/`gzread`/`gzwrite`/`gzprintf`)** — a stdio-like facade that reads and writes `.gz` files directly.

Internally, `deflate.c` performs LZ77 matching via a hash-chain table; the compression level (0–9) tunes how hard the matcher searches (`good_match`, `max_lazy`, `nice_match`, `max_chain` in the level table). `memLevel` and `windowSize` (up to 32 KB) trade memory for ratio. The output is then Huffman-coded in `trees.c`. Decompression (`inflate.c`) is a table-driven state machine that is far simpler and faster than compression. Window management (`windowBits`) selects raw DEFLATE (negative), zlib (positive), or gzip (+16) wrapping — a single numeric argument that quietly controls which of the three formats you get, and a frequent source of confusion.

The code is deliberately portable to a startling range of targets (16-bit DOS, VMS, mainframes) and the inner loops are scalar C. There is no SIMD, no multithreading inside a single stream, and no assembly in mainline. That portability-first posture is the architectural decision every performance fork exists to reverse.

## Production Notes

- **The GitHub default branch is `develop`, not a release.** Tagged releases (`v1.3.1`, etc.) are what you should vendor; `develop` accumulates unreleased fixes. Pulling `HEAD` of the default branch gives you an untagged, mid-flight tree. Releases are also distributed as tarballs from zlib.net, and distro packages often carry additional patches.
- **Security history is real and worth knowing.** CVE-2018-25032 was a memory-corruption bug in `deflate()` when using `memLevel=1`, latent for years and fixed in 1.2.12[^4]. CVE-2022-37434 was a heap buffer over-read in `inflate()` reachable through `inflateGetHeader()` with a crafted gzip extra field, fixed in 1.2.13[^5]. If you accept untrusted compressed input, pin to >= 1.2.13.
- **Decompression is an attack surface.** DEFLATE achieves high ratios, so a small input can expand enormously (a "zip bomb"). zlib will happily inflate it; bounding output size and rejecting oversized streams is the caller's responsibility, not the library's.
- **Thread safety is per-stream, not per-context.** The library is reentrant and thread-safe as long as each thread owns its own `z_stream`; sharing a stream across threads is not supported. See the FAQ for the caveats the README points to.
- **`windowBits` sign/offset conventions bite people.** Compressing with one wrapping and decompressing expecting another silently fails with `Z_DATA_ERROR`. Getting gzip vs zlib vs raw right is a config detail, not an obvious one.
- **Throughput lags modern alternatives by a wide margin.** For CPU-bound pipelines, drop-in forks (zlib-ng) or newer codecs (zstd) are typically several times faster at comparable ratios. zlib's value is compatibility and ubiquity, not speed.

## When to Use / When Not

**Use when:**
- You need to read or write PNG, gzip, or zlib-format data, or speak HTTP `Content-Encoding: gzip` / `deflate` — format compatibility is mandatory and non-negotiable.
- You want a tiny, dependency-free, maximally portable C library that is already present on essentially every platform.
- Long-term format and API stability matter more than raw throughput.

**Avoid when:**
- Compression/decompression throughput is your bottleneck — reach for a SIMD-optimized fork or a modern codec.
- You control both ends and are free to pick a format: zstd or brotli give better ratio-per-CPU.
- You need built-in multithreaded compression of a single stream — zlib does not provide it.

## Alternatives

- zlib-ng/zlib-ng — near-drop-in fork with SIMD-optimized deflate/inflate and a zlib-compatible API; use it when you need zlib's formats but far more speed.
- facebook/zstd — modern codec with better ratio and much higher throughput; use it when you own both ends and aren't bound to DEFLATE.
- google/brotli — strong ratio on text/web assets with a built-in dictionary; use it for HTTP content encoding where clients support `br`.
- google/zopfli — produces smaller DEFLATE/gzip/zlib output than zlib at a large CPU cost; use it for write-once assets where every byte counts.
- lz4/lz4 — extreme speed, lower ratio; use it when latency dominates and you don't need compatibility.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 1995-05 | First public release; DEFLATE library extracted from gzip lineage[^3]. |
| 1.2.0 | 2003-07 | Major internal rewrite of the deflate/inflate engine. |
| 1.2.11 | 2017-01 | Long-lived stable baseline; shipped in countless distros for years. |
| 1.2.12 | 2022-03 | Fixed CVE-2018-25032 (deflate memory corruption at `memLevel=1`)[^4]. |
| 1.2.13 | 2022-10 | Fixed CVE-2022-37434 (inflate heap over-read via gzip header)[^5]. |
| 1.3 | 2023-08 | Cleanup release; build and portability fixes. |
| 1.3.1 | 2024-01 | Security-hardening and minor fixes over 1.3. |
| 1.3.2.1 | 2026 | Current release per the repository README[^3]. |

## References

[^1]: zlib License, the permissive BSD-like grant embedded in the source and README. GitHub's license detector reports NOASSERTION because the text lives in the README rather than a standalone LICENSE file; the SPDX identifier is `Zlib`. https://spdx.org/licenses/Zlib.html
[^2]: RFC 1951 (DEFLATE), with RFC 1950 (zlib) and RFC 1952 (gzip) defining the container formats. https://datatracker.ietf.org/doc/html/rfc1951
[^3]: zlib README and `zlib.h` — copyright (C) 1995–2026 Jean-loup Gailly and Mark Adler; the library contains no third-party code and `zlib.h` is the canonical API documentation. https://github.com/madler/zlib
[^4]: CVE-2018-25032 — memory corruption in `deflate()` with certain `memLevel`/level settings, fixed in zlib 1.2.12. https://nvd.nist.gov/vuln/detail/CVE-2018-25032
[^5]: CVE-2022-37434 — heap buffer over-read in `inflate()` via a large gzip header extra field, fixed in zlib 1.2.13. https://nvd.nist.gov/vuln/detail/CVE-2022-37434

## Tags

c, compression, deflate, gzip, zlib, lossless, data-compression, library, systems, checksum
