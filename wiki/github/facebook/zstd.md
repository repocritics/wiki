# facebook/zstd

> Zstandard — a fast lossless compression algorithm and reference C library, tuned for real-time use at zlib-level ratios and beyond.

[GitHub repo](https://github.com/facebook/zstd) ·
[Official website](https://facebook.github.io/zstd/) ·
[License: BSD-3-Clause OR GPL-2.0](https://github.com/facebook/zstd/blob/dev/LICENSE)

## Overview

Zstandard (`zstd`) is a lossless compression algorithm created by Yann Collet at
Facebook, first released in 2015 and reaching a stable 1.0 format in 2016[^1]. Its
design goal is a better ratio/speed tradeoff than zlib/DEFLATE across the whole
curve: at its default level it compresses noticeably tighter than gzip while both
compressing and — especially — decompressing much faster. Decompression speed stays
roughly constant (~1.5–2 GB/s on a modern core) regardless of the compression level
chosen, which is the property that made it attractive as a drop-in replacement for
gzip in storage and transport layers.

The wire format is stable and standardized as RFC 8478 (2018), later obsoleted by
RFC 8878 (2021)[^2]. This repository is the reference implementation — a C89-friendly
library (`libzstd`) plus a CLI that reads and writes `.zst` and can also handle
`.gz`, `.xz`, and `.lz4`. Because the format is an IETF standard with multiple
independent implementations, `zstd` is safe to adopt as an interchange format, not
just a local tool.

The defining tension is breadth of the speed/ratio dial. A single library spans
negative "faster-than-lz4" levels through level 22 "smaller-than-xz-ish" ultra
modes, plus a dictionary-training subsystem for small records. That flexibility is
the selling point, but it also means a naive `zstd file` (level 3) leaves a large
part of the design space unused, and mis-set levels are the most common source of
"zstd is slow" or "zstd doesn't compress well" complaints.

## Getting Started

```bash
# Debian/Ubuntu: apt install zstd  ·  macOS: brew install zstd
# Or from source (make is the reference build system):
git clone https://github.com/facebook/zstd && cd zstd && make && sudo make install
```

```bash
zstd -19 bigfile.tar            # high ratio  -> bigfile.tar.zst
zstd --fast=4 stream.log        # negative level, faster than lz4
zstd -d bigfile.tar.zst         # decompress
tar --zstd -cf out.tar.zst dir/ # tar integration (GNU tar 1.31+)
```

```c
/* libzstd one-shot API */
#include <zstd.h>
size_t bound = ZSTD_compressBound(srcSize);
size_t n = ZSTD_compress(dst, bound, src, srcSize, /*level=*/3);
if (ZSTD_isError(n)) { /* ZSTD_getErrorName(n) */ }
```

## Architecture / How It Works

`zstd` is an LZ77-family match finder feeding an entropy coder. The pipeline splits
each frame into blocks; within a block, literals and match sequences (offset, match
length, literal length) are separated and entropy-coded independently. Two coders do
the work: **Huff0**, a fast Huffman implementation for literals, and **FSE**, a
tabled-ANS (Asymmetric Numeral System) coder for the sequence symbols[^3]. FSE is
what lets `zstd` approach arithmetic-coding efficiency at Huffman-like speed, and it
originated in Collet's separate FiniteStateEntropy project.

Compression levels select entirely different match-finder strategies rather than just
tuning one. Low/negative levels use a single-probe hash chain; mid levels add lazy
matching; high levels (roughly 17+) switch to a btree/optimal parser that evaluates
candidate parses by cost. This is why decompression cost is level-independent — the
decoder just replays sequences — while compression time varies by orders of
magnitude.

Two subsystems are worth calling out because they change the ratio math:

- **Dictionaries.** `zstd --train samples/* -o dict` builds a dictionary from many
  small samples; compressing/decompressing with `-D dict` then dramatically improves
  the ratio on small records (KBs) that otherwise have no "past" to reference[^1].
  Dictionaries are data-specific and must be shipped to both ends.
- **Long-distance matching (`--long`).** Extends the match window to hundreds of MB,
  useful for large files with far-apart redundancy (VM images, backups). It raises
  memory use on both compress and decompress, and the decoder must be told the window
  size it is allowed to allocate.

The library ships a stable API (`zstd.h`) and an explicitly unstable "experimental"
API guarded by `ZSTD_STATIC_LINKING_ONLY`. Advanced parameters use the newer
`ZSTD_CCtx_setParameter` / `ZSTD_compress2` context API rather than the older
one-shot signatures.

## Production Notes

- **Pick a level deliberately.** Default is 3. Levels 1–5 are the sweet spot for
  hot-path/streaming use; 19 is the practical "archive" ceiling; 20–22 need
  `--ultra` and large windows, cost a lot of RAM, and rarely pay off versus 19.
- **Decompression memory is attacker-controlled via window size.** A malicious frame
  can request a very large window. Long-running services decompressing untrusted
  input should cap it with `ZSTD_d_windowLogMax` (CLI: `--memory=`/`--long=` limits);
  otherwise a small `.zst` can force a large allocation.
- **Multithreading is opt-in.** `-T0` uses all cores; without `-T` the CLI is
  single-threaded. `libzstd` must be built with multithread support (`ZSTD_MULTITHREAD`)
  for the threaded path to exist at all — a common surprise in stripped builds.
- **Streaming needs the streaming API.** `ZSTD_compress`/`decompress` are one-shot and
  hold the whole buffer; for pipes/sockets use `ZSTD_compressStream2` /
  `ZSTD_decompressStream` and honor the returned hint sizes.
- **Format is forward/backward stable, the library ABI mostly is.** Frames written by
  old versions decode on new ones and vice versa within the documented window. The
  stable `zstd.h` API is maintained; code reaching into experimental symbols can and
  does break across minor releases.
- **`.zst` has no built-in encryption or manifest.** It is a compression container,
  not an archive format — no file list, no permissions. Pair with `tar` for archives.

## When to Use / When Not

**Use when:**
- You want a better ratio than gzip with equal-or-faster speed, as a general default.
- You compress many small, similar records and can train a dictionary.
- You need a standardized (RFC 8878) interchange format with broad language bindings.
- You want one knob spanning "faster than lz4" to "denser than gzip" without swapping libraries.

**Avoid when:**
- You need maximum decompression throughput above all and can accept a worse ratio — lz4 decodes faster.
- You need the absolute best ratio and don't care about speed — xz/LZMA and brotli (text/web) can beat high-level zstd on some corpora.
- You need an archive format with metadata/encryption — zstd only compresses a stream.
- Your ecosystem is locked to gzip/DEFLATE at the protocol level and can't negotiate `zstd`.

## Alternatives

- lz4/lz4 — same author; use when decompression speed dominates and ratio is secondary.
- google/brotli — use for HTTP/web text assets where the shared static dictionary and density help.
- google/snappy — use when you want minimal, dependency-light byte-shaving with very high speed and low ratio.
- tukaani/xz (LZMA) — use when you want the tightest ratio for cold archives and can spend the CPU.
- madler/zlib — use only when DEFLATE/gzip compatibility is a hard protocol requirement.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015-01 | First public release by Yann Collet[^1]. |
| 1.0.0 | 2016-08 | Format declared stable; production adoption begins[^1]. |
| 1.3.4 | 2018-03 | Adaptive/multithreaded compression maturing; long-distance matching. |
| — | 2018-10 | Format standardized as RFC 8478[^2]. |
| 1.4.0 | 2019-04 | New advanced context API (`ZSTD_CCtx_setParameter`). |
| 1.5.0 | 2021-05 | Middle-level match-finder rewrite; sizeable speed gains. |
| — | 2021-02 | RFC 8878 obsoletes 8478 as the format spec[^2]. |
| 1.5.7 | 2025 | Current line used in the project's reference benchmarks[^4]. |

## References

[^1]: Zstandard project README and homepage — history, dictionary training, dual BSD/GPLv2 license. https://facebook.github.io/zstd/
[^2]: RFC 8878, "Zstandard Compression and the 'application/zstd' Media Type" (2021), obsoleting RFC 8478 (2018). https://datatracker.ietf.org/doc/html/rfc8878
[^3]: Yann Collet, FiniteStateEntropy (Huff0 + FSE entropy stages). https://github.com/Cyan4973/FiniteStateEntropy
[^4]: facebook/zstd README benchmark table (zstd 1.5.7 on the Silesia corpus via lzbench). https://github.com/facebook/zstd

## Tags

compression, c, lossless, data-compression, cli, library, algorithm, dictionary-compression, rfc8878, entropy-coding, systems, facebook
