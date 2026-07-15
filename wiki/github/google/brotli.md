# google/brotli

> Google's general-purpose lossless compressor — deflate-class speed, denser output, and a built-in dictionary tuned for the web. It is what `Content-Encoding: br` decodes.

[GitHub repo](https://github.com/google/brotli) ·
[Official website](https://brotli.org) ·
[License: MIT](https://github.com/google/brotli/blob/master/LICENSE)

## Overview

Brotli is a lossless compression format and reference C library from Google, defined in [RFC 7932](https://datatracker.ietf.org/doc/html/rfc7932)[^1]. It combines a modern LZ77 variant, Huffman coding, and 2nd-order context modeling. The compression ratio is comparable to the strongest general-purpose methods, while decompression stays close to deflate in speed — which is the point: it was designed to replace gzip on the web without slowing browsers down[^2].

The defining feature is a **built-in static dictionary** of roughly 13,000 common strings — HTML tags, CSS keywords, JS fragments, and words across several languages — baked into the format. On small text payloads (which dominate web traffic) this dictionary gives brotli a ratio advantage that stream-only compressors cannot match without training. The tradeoff: on binary or already-compressed data the dictionary contributes nothing and the asymmetric cost profile becomes pure overhead.

The cost that shapes every deployment is **asymmetry**. Encoding at the maximum quality level is extremely slow; decoding is fast and runs in bounded memory. That is fine for content compressed once and served many times (static assets, fonts), and a footgun for anything compressed on the fly. GitHub labels the repo TypeScript because of the JS binding tree, but the core (`libbrotlienc`, `libbrotlidec`, `libbrotlicommon`) is C89-portable C.

## Getting Started

```bash
# CLI (Debian/Ubuntu, Homebrew, or vcpkg)
apt install brotli        # or: brew install brotli
brotli -q 11 index.html   # -> index.html.br
brotli -d index.html.br   # decompress

# Python module
pip install brotli
```

```python
import brotli

data = open("page.html", "rb").read()
compressed = brotli.compress(data, quality=11)   # 0..11, slow at 11
restored = brotli.decompress(compressed)
assert restored == data
```

```c
/* C: one-shot decode. Link -lbrotlidec -lbrotlicommon */
#include <brotli/decode.h>
size_t out_size = OUT_CAPACITY;
BrotliDecoderResult r = BrotliDecoderDecompress(
    in_size, in_buf, &out_size, out_buf);
/* r == BROTLI_DECODER_RESULT_SUCCESS on complete input */
```

## Architecture / How It Works

The format layers three classic techniques, then adds the dictionary:

1. **LZ77 with a large window** — back-references into previously seen data. Window size is `lgwin` 10–24, i.e. up to a 16 MB sliding window. Larger windows find more matches at the cost of decoder memory.
2. **Context modeling** — a 2nd-order context mixes the previous two bytes to select among literal/distance code contexts, which is where much of brotli's edge over deflate on structured text comes from.
3. **Huffman coding** — per-block prefix codes, with the block structure itself chosen by the encoder.
4. **Static dictionary** — the ~120 KB embedded dictionary plus a set of transforms (case changes, suffixes) that let one dictionary entry match many surface forms.

The library splits into three shared objects: `libbrotlicommon` (shared tables and the dictionary), `libbrotlienc` (encoder), and `libbrotlidec` (decoder). Applications that only decompress link the decoder alone and avoid the heavier encoder tables.

**Quality is the master dial.** Levels 0–11 trade encode time for ratio. Levels 0–4 are roughly deflate-speed; 5–9 are the practical middle; 10–11 apply exhaustive match search and cost an order of magnitude more CPU for a few percent more density. Decode speed and decode memory are almost independent of the quality used to encode — they depend on `lgwin`, not `quality`.

Per the README, brotli is a **raw stream format**: no checksum, no stored length, no magic-number framing. Editing arbitrary ranges of a compressed stream produces output the decoder will happily accept as valid. Integrity and framing are explicitly someone else's job (TLS, HTTP framing, or an outer container).

## Production Notes

**Never run quality 11 on the request path.** Dynamic compression of responses should sit around quality 4–5, where brotli is competitive with gzip on both speed and ratio. Reserve 10–11 for build-time precompression of static assets, where the slow encode happens once. Serving a pre-built `.br` alongside the original is the standard nginx/Apache/CDN pattern; the `ngx_brotli` module and most CDNs support "serve precompressed if present."[^3]

**Browsers only accept `br` over HTTPS.** `Content-Encoding: br` is advertised in `Accept-Encoding` only on secure origins in every major browser. Do not expect brotli negotiation over plain HTTP.

**Window size drives decoder memory.** A `lgwin` of 24 lets the decoder allocate on the order of the window plus ring-buffer state per stream. On highly concurrent servers or memory-constrained clients this multiplies; pick the smallest window that still hits your ratio target rather than defaulting to the maximum.

**The dictionary helps text, not blobs.** Brotli's advantage over zstd is largest on small HTML/CSS/JS. On large binaries, images, or already-compressed payloads, zstd is usually faster for equal or better ratio, and brotli's high-quality encode cost is wasted.

**Streaming APIs are stateful and easy to misuse.** `BrotliEncoderCompressStream` / `BrotliDecoderDecompressStream` require draining output until the operation reports it wants more input; treating them like one-shot calls silently truncates. Prefer the one-shot `*Decompress` helpers unless you genuinely stream.

**Upgrades are conservative.** The format is frozen by RFC 7932; the library has kept a stable ABI across 1.0.x. The 1.1.0 additions (large-window and shared-dictionary modes) are opt-in and gated behind explicit encoder flags, so existing decoders keep working[^4]. Releases are infrequent — treat a quiet changelog as stability, not abandonment. The project is fuzzed continuously via OSS-Fuzz.

## When to Use / When Not

**Use when:**
- You serve static web assets (HTML, CSS, JS, SVG, JSON) and can precompress at build time.
- You compress WOFF2 fonts or other text-heavy artifacts once and ship them widely.
- You need output a browser can decode natively without a JS shim.
- Decode speed and bounded decode memory matter more than encode speed.

**Avoid when:**
- You must compress on the fly at high ratio under latency budget — quality 11 is far too slow; consider a lower quality or zstd.
- Your data is binary/already-compressed, where the dictionary and context model earn little.
- You need built-in integrity or length framing — brotli provides neither; wrap it.
- You want a tunable speed/ratio curve for backups or logs at scale — zstd's range is wider.

## Alternatives

- facebook/zstd — use instead when you need a wide, tunable speed/ratio curve, trainable dictionaries, or fast compression at scale; brotli still edges it on max-ratio small web text.
- madler/zlib — use instead when universal compatibility (gzip/deflate everywhere) outweighs ratio.
- lz4/lz4 — use instead when decompression throughput is the whole game (real-time, in-memory pipelines) and ratio barely matters.
- google/snappy — use instead when you want very fast compress+decompress for RPC/storage internals and accept a weaker ratio.
- Mark Adler's independent decoder (madler/brotli) — use instead when you want a minimal, spec-only decoder without Google's full library.

## History

| Version | Date | Notes |
|---------|------|-------|
| WOFF2 origin | 2013 | Format grows out of Google's web-font compression work[^2]. |
| RFC 7932 | 2016-07 | Brotli Compressed Data Format standardized by IETF[^1]. |
| 1.0.0 | 2016-10 | Stable encoder/decoder C API. |
| 1.0.9 | 2020-08 | Long-lived 1.0.x line; broad distro adoption. |
| 1.1.0 | 2023-08 | Large-window and shared-dictionary modes; latest release[^4]. |

## References

[^1]: J. Alakuijala, Z. Szabadka, "Brotli Compressed Data Format" — RFC 7932, IETF, 2016-07. https://datatracker.ietf.org/doc/html/rfc7932
[^2]: Google Open Source Blog, "Introducing Brotli: a new compression algorithm for the internet" — 2015-09-22. https://opensource.googleblog.com/2015/09/introducing-brotli-new-compression.html
[^3]: `ngx_brotli` — official nginx module for serving/compressing brotli. https://github.com/google/ngx_brotli
[^4]: google/brotli releases. https://github.com/google/brotli/releases

## Tags

c, compression, lossless, http, web-performance, brotli, rfc-7932, lz77, huffman, content-encoding, cli-tool
