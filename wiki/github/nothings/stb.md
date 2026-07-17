# nothings/stb

> A collection of single-file, public-domain C/C++ libraries — copy one header into your project and use it, no build system required.

[GitHub repo](https://github.com/nothings/stb) ·
[Official website](https://nothings.org) ·
License: Public Domain or MIT (dual; GitHub reports the license as `NOASSERTION`)

## Overview

`stb` is Sean T. Barrett's collection of 21 self-contained C headers covering
image decoding, font rasterization, image resizing, audio decoding, typesafe
containers, and a scattering of game-dev and parsing utilities[^1]. Each file
stands alone: there is no `libstb`, no package to link, no configure step. You
drop the `.h` into your source tree, define one macro in exactly one translation
unit, and `#include` it. That distribution model — not any single algorithm — is
what made `stb` ubiquitous. `stb_image.h` and `stb_truetype.h` in particular are
vendored into a large fraction of C/C++ game engines, tools, and hobby projects,
and are dependencies of widely-used libraries such as Dear ImGui.

The defining tradeoff is convenience versus safety and performance. `stb`
libraries are optimized for "easy to integrate, easy to read, easy to ship,"
explicitly at the expense of being potentially "less featureful, slower, and/or
use more memory" than dedicated alternatives[^2]. The maintainers say so
directly: if you already use libpng/libjpeg/FreeType, there is usually no reason
to switch. The most consequential consequence of this philosophy is security —
the project's own README opens with a warning that security-relevant bugs are
discussed in public and fixes may take significant time to land[^3].

The code is dual-licensed: public domain (per the Unlicense spirit) or MIT, your
choice, declared in every file. There is no attribution requirement, though the
author appreciates it. This is a deliberate stance — the repository ships a
`why_public_domain.md` explaining the preference over GPL/BSD/zlib[^4].

## Getting Started

There is nothing to install. Fetch the header you want and commit it into your
project (vcpkg, Conan, and various distro packages also carry `stb`, but vendoring
the single file is the intended path). Define the implementation macro in exactly
one `.c`/`.cpp` file:

```c
// In ONE translation unit, before the include:
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"

int main(void) {
    int w, h, channels;
    // desired_channels = 4 forces RGBA
    unsigned char *pixels = stbi_load("photo.png", &w, &h, &channels, 4);
    if (!pixels) {
        // stbi_failure_reason() returns a human-readable string
        return 1;
    }
    // ... use pixels (w * h * 4 bytes) ...
    stbi_image_free(pixels);
    return 0;
}
```

Every other file that only *uses* the API includes the same header *without*
defining the macro, so it acts as a plain declaration header. The correct macro
name is documented at the top of each file (e.g. `STB_TRUETYPE_IMPLEMENTATION`,
`STB_DS_IMPLEMENTATION`).

## Architecture / How It Works

The single-file-header pattern is the whole architecture. A header does double
duty: with no macro defined it emits only declarations; with `STB_X_IMPLEMENTATION`
defined it also emits the function bodies. Defining the macro in two translation
units within one link target produces duplicate-symbol errors — a common
first-time footgun. The rationale is Windows-centric: Windows lacks a standard
place for libraries to live and suffers from runtime-mismatch link conflicts, so
shipping code as headers that compile straight into the consumer sidesteps both
problems[^5]. The choice of one file over two (header + source) is deliberate —
one file is trivial to attach, drop in, or update.

Most libraries are mutually independent, so you can take just the one you need.
There are a few intentional pairings: `stb_truetype.h` rasterizes glyphs, and
`stb_rect_pack.h` packs them into a texture atlas — together they bake font
atlases without FreeType. `stb_textedit.h` supplies only the *guts* of a text
editor (cursor movement, selection, undo) and expects the host to provide
rendering and the string backing store.

Some engineering constraints follow directly from the header model.
`stb_image.h` will use SSE2 only if you compile with `-msse2`; it will not do
runtime CPU-feature detection, because GCC's supported path for that requires
multiple source files, which a one-source-file header cannot provide[^6]. Error
handling in the image path uses `setjmp`/`longjmp`. The libraries are written to
a conservative, old-compiler-friendly C dialect (the author has noted working in
very old MSVC), avoiding C99-isms in places — which is why you occasionally see
non-idiomatic constructs.

## Production Notes

**Security is the headline caveat, and it is the maintainers' own framing.**
`stb_image.h` decodes attacker-controllable, compressed binary formats (PNG, JPEG,
GIF, PSD, HDR, and more) in plain C with manual memory management. Historically it
has been a recurring source of memory-corruption bugs, and the project explicitly
states that security fixes are handled in the open and may be slow[^3]. Do not
feed untrusted images to `stb_image` in a security-sensitive process without
sandboxing, resource limits, and fuzzing. For hardened untrusted-input decoding,
memory-safe decoders (see Alternatives) are the safer choice.

**No new image formats will be added.** The maintainers closed the door on
expanding `stb_image`'s format list specifically to limit the attack surface that
must be secured[^7]. If you need AVIF, WebP, or JPEG XL, `stb_image` is not your
path.

**Vendoring means you own updates.** Because you copy the header in, you do not
get security patches automatically. Track the upstream file and re-vendor
periodically; pin the version comment at the top of each header so you know what
you have. There are no coordinated repo-wide releases — each library carries its
own version number and advances independently.

**Performance is "good enough," not best-in-class.** `stb_image` is slower than
libjpeg-turbo on JPEG and than a tuned libpng on PNG. `stb_image_resize2.h`
(a from-scratch rewrite by Jeff Roberts replacing the original resizer) is
substantially faster and higher quality than its predecessor and is the one to
use for new code. `stb_ds.h` uses macro-heavy generic containers whose ergonomics
some teams dislike and whose compile-error messages can be opaque.

**Thread-safety varies per library and is not uniformly documented.** The image
decoders are generally reentrant on distinct buffers, but several libraries expose
global configuration flags (e.g. `stbi_set_flip_vertically_on_load`) that are
process-global state; set them once at startup, not per-thread.

## When to Use / When Not

**Use when:**
- You want to load an image or rasterize a font with zero build-system friction.
- You are prototyping, writing a game/tool, or working in an environment where
  adding a linked dependency is painful (Windows, embedded, single-binary tools).
- Public-domain / no-attribution licensing matters to you.
- The input is trusted, or you can sandbox/fuzz the decode path yourself.

**Avoid when:**
- You decode untrusted images in a security-critical service and cannot sandbox —
  prefer a memory-safe or hardened decoder.
- You need maximum decode/encode throughput (use libjpeg-turbo, a tuned libpng).
- You need modern formats (WebP/AVIF/JXL) or a full-featured font engine with
  hinting, shaping, and color fonts (use FreeType + HarfBuzz).

## Alternatives

- libjpeg-turbo/libjpeg-turbo — use instead when JPEG decode/encode throughput
  matters; SIMD-optimized and battle-tested, at the cost of a linked dependency.
- glennrp/libpng — use instead when you need the reference PNG implementation with
  full chunk support and long-term security maintenance.
- google/wuffs — use instead when you decode untrusted images and want
  memory-safety guarantees by construction (formats transpile to safe C).
- freetype/freetype — use instead of `stb_truetype` when you need hinting,
  subpixel rendering, and (with HarfBuzz) complex-script text shaping.
- lvandeve/lodepng — use instead when you want a single-file PNG-only codec with a
  simpler, more auditable surface than `stb_image`.

## History

`stb` predates its GitHub repository; several libraries (notably `stb_image`)
circulated for years before the repo was published. There is no unified release
cadence — each header versions independently, so the table below records
repository milestones and a snapshot of current per-library versions rather than
project-wide releases.

| Item | Date / Version | Notes |
|------|----------------|-------|
| Repository published on GitHub | 2014-05-25 | Consolidated Sean Barrett's headers into `nothings/stb`.[^8] |
| `stb_image.h` | 2.30 | JPG/PNG/TGA/BMP/PSD/GIF/HDR/PIC decode; no new formats planned.[^1] |
| `stb_truetype.h` | 1.26 | TrueType parse + rasterize.[^1] |
| `stb_image_write.h` | 1.16 | PNG/TGA/BMP write.[^1] |
| `stb_image_resize2.h` | 2.18b | Rewrite (Jeff Roberts) superseding the original resizer.[^1] |
| `stb_ds.h` | 0.67 | Typesafe dynamic arrays and hash tables.[^1] |
| `stb_vorbis.c` | 1.22 | Ogg Vorbis decoder (only non-header file in the set).[^1] |
| Last commit to `master` | 2026-04-15 | Repository remains maintained but low-churn.[^8] |

## References

[^1]: `nothings/stb` README — library table and per-library versions. https://github.com/nothings/stb/blob/master/README.md
[^2]: `nothings/stb` README FAQ, "Are they better somehow?" https://github.com/nothings/stb#faq
[^3]: `nothings/stb` README security notice ("discusses security-relevant bugs in public…"). https://github.com/nothings/stb/blob/master/README.md
[^4]: "Why public domain?" https://github.com/nothings/stb/blob/master/docs/why_public_domain.md
[^5]: `nothings/stb` README FAQ, "Why single-file headers?" https://github.com/nothings/stb#faq
[^6]: `nothings/stb` README FAQ, SSE support in GCC; see issues #280 and #410. https://github.com/nothings/stb/issues/280
[^7]: `nothings/stb` README FAQ, "Will you add more image types to stb_image.h?" https://github.com/nothings/stb#faq
[^8]: GitHub repository metadata, `nothings/stb` (created 2014-05-25, latest push 2026-04-15). https://github.com/nothings/stb
[^9]: `stb_howto.txt` — guidance on writing single-file libraries. https://github.com/nothings/stb/blob/master/docs/stb_howto.txt

## Tags

c, cpp, single-file-header, image-decoding, truetype-fonts, public-domain, game-dev, no-dependencies, image-resize, ogg-vorbis
