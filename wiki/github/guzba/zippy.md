# guzba/zippy

> Pure Nim implementation of DEFLATE, zlib, gzip, zip, and tar — no C dependencies, competitive with zlib on speed.

[GitHub repo](https://github.com/guzba/zippy) ·
[API reference](https://guzba.github.io/zippy/) ·
[License: MIT](https://github.com/guzba/zippy/blob/master/LICENSE)

## Overview

Zippy is a compression library for Nim that implements the DEFLATE (RFC 1951), zlib (RFC 1950), and gzip (RFC 1952) formats from scratch, plus reading of ZIP archives and tarballs (.tar, .tar.gz, .tgz)[^1]. Its explicit design goal is to be small, fast, and dependency-free: unlike the official `nim-lang/zip` package, which binds to C zlib and libzip, Zippy links nothing external. That matters in Nim because cross-compilation and static binaries are common use cases, and dragging in a system zlib is a frequent source of Windows/macOS build friction.

The trade the library makes is reimplementation risk in exchange for build simplicity. The author mitigates this with a cross-implementation validation suite (`tests/validate.nim` round-trips data against other DEFLATE implementations) and fuzzing (`tests/fuzz.nim`, `tests/stress.nim`)[^1]. The project is by Ryan Oldenburg (guzba), whose sibling libraries — Pixie (2D graphics) and Mummy (HTTP server) — consume Zippy for PNG zlib streams and HTTP gzip respectively, so the code is exercised more widely than its 282-star count suggests.

Maintenance is slow-cadence but real: created October 2020, latest tagged release 0.10.19 (March 2026), last push May 2026[^2]. It is a mature library in maintenance mode rather than an actively evolving one — 0.10.x has been the series since mid-2022.

## Getting Started

```bash
nimble install zippy
```

Compress and decompress in-memory data:

```nim
import zippy

let compressed = compress("Hello, compression!", BestCompression, dfGzip)
let original = uncompress(compressed, dfGzip)
assert original == "Hello, compression!"
```

Extract a ZIP archive or tarball:

```nim
import zippy/ziparchives

extractAll("archive.zip", "output_dir/")
```

The `dfDetect` data format sniffs zlib vs gzip headers on decompression, which is convenient for HTTP `Content-Encoding` handling (see the `examples/` folder for HTTP client/server gzip usage)[^1].

## Architecture / How It Works

Zippy is a from-scratch DEFLATE codec: LZ77 match finding with hash chains feeding canonical Huffman coding, with the standard compression-level dial trading match-search effort for ratio (`BestSpeed` through `BestCompression`, mirroring zlib's level semantics). Decompression validates the adler32 (zlib) or crc32 (gzip) checksums the wrappers require. Everything operates on Nim strings/seqs in memory — there is no incremental streaming interface equivalent to zlib's `z_stream`, which is the most important architectural fact about the library (see Production Notes).

The archive layer sits on top: `zippy/ziparchives` parses the ZIP central directory and inflates entries (deflate and store methods), and `zippy/tarballs` handles tar with optional gzip. Reading/extraction is the mature path; archive creation is supported but historically narrower in scope than extraction, and the archive APIs were reworked during the 0.10 series.

Portability is a first-class concern: the library is tested under Nim's default GC and `--gc:arc`/`--gc:orc`, under both `nim c` and `nim cpp`, and with MSVC (`--cc:vcc`) on Windows[^1]. This makes it a safe dependency for libraries that cannot dictate their consumers' toolchain — which is exactly how Pixie uses it.

## Production Notes

- **No streaming API.** `compress`/`uncompress` take and return whole buffers. Decompressing a multi-gigabyte gzip file means holding both input and output in memory. If you need bounded-memory streaming (log pipelines, proxies), Zippy is the wrong shape; bind C zlib or zstd instead.
- **Untrusted input.** The codec has been fuzzed against crashes[^1], but DEFLATE's ~1000:1 amplification still applies: a small malicious payload can expand to a huge allocation. Enforce your own size limits before decompressing untrusted data — the whole-buffer API gives you no natural backpressure point.
- **Performance is genuinely good, but self-reported.** The README's benchmarks (Ryzen 5 5600X) show roughly 1.5–2x faster than stock-Linux zlib on both compress and uncompress[^3]. These are the author's numbers against `nim-lang/zip`; run `tests/benchmark.nim` on your own workload. The baseline is stock zlib, not zlib-ng or libdeflate, which narrow or reverse the gap.
- **ZIP feature coverage.** Expect the common cases (deflate/store entries) to work. Encrypted ZIPs and exotic compression methods are not supported; if you ingest arbitrary user-uploaded archives, validate against your corpus first.
- **Bus factor and cadence.** Effectively a single-maintainer project with months-long release gaps (0.10.16 in Aug 2024 → 0.10.17 in Dec 2025)[^2]. The library is small and stable enough that vendoring is a realistic fallback. Still 0.x after five years; the API has been stable through 0.10.x, but the archive-module rework earlier in 0.x broke consumers once.

## When to Use / When Not

**Use when:**
- You are writing Nim and want gzip/zlib/deflate without any C toolchain or system-library dependency (static binaries, cross-compilation, Windows).
- You need HTTP gzip encoding/decoding in a Nim service or client.
- You need to read ZIP/tar archives whose contents fit comfortably in memory.
- You are already in the guzba/treeform ecosystem (Pixie, Mummy) — it is the native choice there.

**Avoid when:**
- You need streaming compression with bounded memory — whole-buffer only.
- You need maximum-trust decompression of hostile input at scale; C zlib, zlib-ng, or libdeflate have vastly larger deployment and audit surface.
- You need modern formats (zstd, brotli, xz) — Zippy is DEFLATE-family only.
- You need full ZIP feature coverage (encryption, ZIP64 edge cases, uncommon methods) for arbitrary third-party archives.

## Alternatives

- nim-lang/zip — official Nim wrapper over C zlib/libzip; use it when you want the battle-tested C implementation and can tolerate the external dependency.
- madler/zlib — the reference C implementation; bind it directly when ecosystem trust and streaming matter more than pure-Nim builds.
- richgel999/miniz — single-file C DEFLATE+ZIP; a comparable "easy to embed" choice for C/C++ codebases rather than Nim.
- facebook/zstd — different format entirely; use when you control both ends of the pipe and want better ratio-speed tradeoffs than DEFLATE.
- guzba/supersnappy — same author, pure Nim Snappy; use for speed-over-ratio internal data paths where format compatibility with gzip/zlib is not required.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-10 | Repo created; pure-Nim deflate/zlib/gzip[^2]. |
| 0.10.0 | 2022-06 | Start of the long-running 0.10.x series[^2]. |
| 0.10.16 | 2024-08 | Last release before a ~16-month gap[^2]. |
| 0.10.17–18 | 2025-12 | Maintenance releases[^2]. |
| 0.10.19 | 2026-03 | Latest tagged release[^2]. |
| — | 2026-05 | Last push: macOS `nim cpp` compatibility fix[^2]. |

## References

[^1]: Zippy README — formats, goals, validation, fuzzing, GC/compiler support. https://github.com/guzba/zippy#readme
[^2]: GitHub releases and commit history for guzba/zippy. https://github.com/guzba/zippy/releases
[^3]: Zippy README, "Performance" — author-run benchmarks vs nim-lang/zip on Ryzen 5 5600X. https://github.com/guzba/zippy#performance

## Tags

nim, compression, deflate, gzip, zlib, zip-archive, tarball, lz77, huffman-coding, library
