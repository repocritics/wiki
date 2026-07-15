# lz4/lz4

> Byte-oriented LZ77 compression tuned for speed over ratio — compresses at ~500+ MB/s per core and decompresses at multiple GB/s.

[GitHub repo](https://github.com/lz4/lz4) ·
[Official website](http://www.lz4.org) ·
License: BSD-2-Clause (library) / GPL-2.0 (CLI tools)

## Overview

LZ4 is a lossless compression algorithm and reference C implementation by Yann Collet (@Cyan4973), who later authored Zstandard[^1]. It sits at the far "fast" end of the compression tradeoff space: it deliberately gives up ratio to keep both compression and, especially, decompression close to memory bandwidth. On the maintainers' reference benchmark (Silesia corpus, single core), the default level compresses the corpus ~2.1× at ~780 MB/s and decompresses at ~4970 MB/s[^2] — roughly an order of magnitude faster to decode than zlib, at a noticeably worse ratio.

The defining tension is ratio versus throughput, and LZ4 picks throughput almost every time. It is the compressor you reach for when the alternative to compressing is not "compress harder" but "don't compress at all" — in-memory caches, network RPC framing, columnar database blocks, kernel swap pages, and filesystem transparent compression. Where a few percent of size matters more than latency, zstd or zlib is the better tool; LZ4 is designed to be invisible in a hot path.

LZ4 is widely embedded rather than run as a program. It ships inside the Linux kernel (zram, zswap, squashfs, and historically btrfs), ZFS/OpenZFS transparent compression, database and storage engines, Hadoop, and countless language runtimes via ports[^3]. The GitHub repository is the canonical C source; the `dev` branch is the default, and releases are cut as `v1.x.y` tags.

## Getting Started

```bash
# Debian/Ubuntu: library + CLI
apt-get install liblz4-dev lz4
# macOS
brew install lz4
# from source
git clone https://github.com/lz4/lz4 && cd lz4 && make && sudo make install
```

Command line, using the interoperable frame format:

```bash
lz4 file.tar            # -> file.tar.lz4
lz4 -d file.tar.lz4     # decompress
lz4 -9 file.tar         # LZ4_HC, higher ratio, same decode speed
```

Minimal C using the one-shot block API:

```c
#include <lz4.h>
#include <string.h>

const char *src = "the quick brown fox the quick brown fox";
int srcSize = (int)strlen(src) + 1;
int bound   = LZ4_compressBound(srcSize);
char *cmp   = malloc(bound);

int cmpSize = LZ4_compress_default(src, cmp, srcSize, bound);

char *out = malloc(srcSize);
/* always decompress_safe on untrusted input: it is bounds-checked */
int outSize = LZ4_decompress_safe(cmp, out, cmpSize, srcSize);
```

## Architecture / How It Works

There are two distinct on-disk representations, and confusing them is the most common integration mistake.

**Block format**[^4] is the raw algorithm output: a sequence of tokens. Each token byte carries two 4-bit nibbles — a literals length and a match length. It is followed by any length-overflow bytes, the literal bytes themselves, a 2-byte little-endian match offset, and then more overflow bytes for the match length. Matches must be at least 4 bytes; the offset is 16 bits, so the back-reference window is capped at 64 KB. A raw block carries no length header, no checksum, and no framing — the decoder must be told the sizes out of band. This is what you get from `LZ4_compress_default` and what most in-memory embedders use.

**Frame format**[^5] wraps blocks for files and streams: a magic number (`0x184D2204`), a frame descriptor, independently or linked blocks, optional per-block and content xxHash checksums, optional content size, and optional dictionary ID. This is the interoperable format the `lz4` CLI and `liblz4`'s `lz4frame.h` produce, and the only one two independent implementations are expected to agree on.

The compressor is a single-pass LZ77 matcher over a hash table of recent 4-byte sequences. The `acceleration` parameter skips more input between hash probes to trade ratio for speed. **LZ4_HC** is a separate, slower encoder using optimal/chain parsing to find better matches; crucially it emits the *same* block format, so HC-compressed data decodes at full LZ4 speed. All levels share one decoder.

**Streaming** (`LZ4_stream_t`, `LZ4_loadDict`, ring buffers) lets consecutive blocks reference the previous 64 KB, recovering ratio on data split into many small blocks. Dictionary compression feeds a fixed prefix (only the final 64 KB is used) to warm the window for small payloads.

## Production Notes

**The dual license is a real trap.** The library under `lib/` is BSD-2-Clause; the command-line programs under `programs/` are GPL-2.0. GitHub reports the repository license as "NOASSERTION" for exactly this reason. Vendoring `lib/lz4.c`/`lz4.h` into a closed-source product is fine; shipping the CLI binary or its source is not, without accepting GPL terms. Audit which files you actually pull in.

**`LZ4_decompress_fast` is a footgun and is deprecated.** It trusts the caller-supplied original size and does no input bounds checking, so malformed or malicious input can overflow the output buffer. Always use `LZ4_decompress_safe` on anything you did not compress yourself in the same process. The fast variant remains only for trusted, size-known hot paths.

**Ratio is structurally limited.** The 64 KB window and 4-byte minimum match mean LZ4 will never approach zstd/zlib ratios on text or redundant data. If you find yourself reaching for LZ4_HC -12 to close the gap, you are usually better served by `zstd -1` or `-2`, which beats HC on both ratio and often speed at low levels.

**Block-format interop requires discipline.** Because raw blocks are unframed, you must transmit the uncompressed size (and ideally a checksum) yourself. A great deal of "LZ4 corruption" reported downstream is actually mismatched framing conventions between a producer using raw blocks and a consumer expecting the frame format.

**ABI and versioning.** `liblz4` maintains a stable soname and API; upgrades within the 1.x line are drop-in, and the bitstream formats are frozen and forward/backward compatible. xxHash is bundled (vendored), so the frame checksum path has no external dependency. Multi-threading is the caller's responsibility — the core is single-threaded and reentrant, so parallelism is achieved by compressing independent blocks/frames concurrently.

## When to Use / When Not

**Use when:**
- Decompression latency dominates and data is read far more than written (caches, DB read paths, swap).
- You need line-rate or memory-bandwidth throughput and can accept a ~2× ratio.
- You are compressing in a kernel, embedded, or real-time context where a small, dependency-free C core matters.
- You want a stable, frozen bitstream to persist for years.

**Avoid when:**
- Storage or bandwidth cost dominates and CPU is cheap — use zstd or zlib for meaningfully smaller output.
- You are tuning HC to high levels to chase ratio — that is zstd's job.
- You need parallel, dictionary-trained, or high-ratio adaptive compression out of the box.

## Alternatives

- facebook/zstd — same author; use instead when you want a better ratio/speed frontier, trained dictionaries, and multithreading, and can spend a little more decode time.
- google/snappy — use instead when you specifically want Google's ecosystem/format; comparable speed, generally worse ratio than LZ4.
- google/brotli — use instead for static web assets where high ratio and a shared dictionary matter and decode speed is secondary.
- nemequ/lzbench — not a codec; use to benchmark LZ4 against alternatives on your own data before committing.
- lz4/lz4 vs zlib/zlib — use zlib when broad legacy interop (gzip/DEFLATE) is the requirement rather than speed.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011 | First public release (originally hosted on Google Code) by Yann Collet[^1]. |
| 1.7.x | 2016 | Frame format maturation, dictionary support. |
| 1.8.x | 2017–2018 | CLI and frame-format refinements. |
| 1.9.0 | 2019 | Reference-benchmark baseline; performance and API cleanup[^2]. |
| 1.9.4 | 2022-08 | Bug/security fixes, build and portability improvements. |
| 1.10.0 | 2024-07 | Latest stable line; continued hardening and tooling updates. |

## References

[^1]: LZ4 project homepage and author (Yann Collet, @Cyan4973). http://www.lz4.org
[^2]: LZ4 README benchmark, Silesia corpus, Core i7-9700K, single core (lzbench). https://github.com/lz4/lz4#benchmarks
[^3]: LZ4 packaging and known ports/embedders. https://repology.org/project/lz4/versions
[^4]: LZ4 block format specification. https://github.com/lz4/lz4/blob/dev/doc/lz4_Block_format.md
[^5]: LZ4 frame format specification. https://github.com/lz4/lz4/blob/dev/doc/lz4_Frame_format.md

## Tags

c, compression, lossless, lz77, data-compression, algorithm, streaming, systems, performance, cli-tool
