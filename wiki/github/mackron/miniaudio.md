# mackron/miniaudio

> Audio playback and capture for C/C++ in a single source file — device I/O,
> decoding, mixing, and 3D spatialization with zero external dependencies.

[GitHub repo](https://github.com/mackron/miniaudio) ·
[Official website](https://miniaud.io) ·
[License: Unlicense OR MIT-0](https://github.com/mackron/miniaudio/blob/master/LICENSE)

## Overview

miniaudio is a cross-platform audio library written in C89-compatible C by
David Reid (also the author of the dr_wav / dr_flac / dr_mp3 decoders, which
are amalgamated into it)[^1]. It began life in 2016 as `mini_al`, a low-level
device abstraction, and was renamed miniaudio at v0.9 (2019)[^2]. Since
v0.11.0 (2021) it spans two layers: a low-level callback API over the raw
audio device, and a high-level engine (`ma_engine` / `ma_sound`) with a node
graph for mixing and effects, a resource manager for streaming files, and
optional 3D spatialization[^3]. The whole library ships as one ~4 MB header
plus a thin `miniaudio.c`, and compiles with no dependencies beyond libc —
platform backends are loaded at runtime via `dlopen`/`LoadLibrary`, so on
Windows and macOS you link against nothing.

The defining tradeoff is scope-in-one-file versus modularity. You get device
I/O, WAV/FLAC/MP3 decoding, resampling, channel mapping, filters, and a game
audio engine from a single translation unit — which is why raylib builds its
audio module on miniaudio[^4] — but you accept a large TU, transparent
caller-allocated structs instead of opaque handles, and a single-maintainer
project. The numbers say the model works: roughly 7,000 stars is top of its
niche for a C audio library, and 5 open issues against a July 2026 push date
indicates unusually disciplined triage for a solo project.

## Getting Started

Vendor the two files into your source tree (the recommended integration —
ABI stability is explicitly not guaranteed, so shared-library builds are
discouraged[^1]):

```bash
# add miniaudio.h + miniaudio.c to your project, then:
cc main.c miniaudio.c -lpthread -lm -o app   # Linux/BSD; nothing extra on Windows/macOS
```

```c
#include "miniaudio.h"
#include <stdio.h>

int main(void)
{
    ma_engine engine;
    if (ma_engine_init(NULL, &engine) != MA_SUCCESS) return -1;

    ma_engine_play_sound(&engine, "sound.wav", NULL);   /* fire-and-forget */

    printf("Press Enter to quit...");
    getchar();
    ma_engine_uninit(&engine);
    return 0;
}
```

On iOS, compile the implementation as Objective-C. Historically the library
was header-only via `MINIAUDIO_IMPLEMENTATION`; that still works in 0.11.x,
but `miniaudio.c` (added in v0.11.22) is the forward-compatible path — the
planned v0.12 drops the single-header model entirely[^5].

## Architecture / How It Works

The stack, bottom to top:

1. **Backends** — WASAPI, DirectSound, WinMM, Core Audio, ALSA, PulseAudio,
   JACK, sndio, audio(4), OSS, AAudio, OpenSL|ES, Web Audio (Emscripten),
   plus Null and a pluggable custom-backend interface[^6]. Backends are
   probed at runtime in priority order; symbols are resolved dynamically so
   the binary carries no hard link-time dependency on any of them.
2. **Device layer** — `ma_device` with a `dataCallback` invoked on a
   high-priority audio thread. miniaudio inserts format/channel/rate
   conversion between your callback and whatever the device natively wants,
   so the callback always sees the format you configured.
3. **Data sources** — a vtable-based `ma_data_source` abstraction unifying
   decoders (WAV/FLAC/MP3 built in via embedded dr_libs), waveform/noise
   generators, ring buffers, and user implementations.
4. **Node graph** — a pull-model DAG of processing nodes (splitters,
   filters, delay, spatializer) driving mixing in the high-level engine.
5. **Engine** — `ma_engine`/`ma_sound` wrap the node graph plus a resource
   manager that decodes on worker threads and streams large files[^3].

Two idioms permeate the API: a config/init pattern (`ma_*_config_init()`
then `ma_*_init()`), and transparent structures — you allocate every object
yourself, on stack or heap, and its address must remain stable for its
lifetime[^1]. There is no hidden allocation at the device layer, which is
why the same code runs on desktop, mobile, and Emscripten.

Nearly every subsystem can be compiled out (`MA_NO_DECODING`,
`MA_NO_DEVICE_IO`, `MA_NO_ENGINE`, etc.), so the low-level-only footprint is
far smaller than the headline file size suggests.

## Production Notes

**The audio thread is yours to ruin.** The data callback runs on a
real-time-priority thread; miniaudio does not police what you do in it.
Allocation, locks, or file I/O in the callback cause glitches under load.
The engine API sidesteps this by doing decode work on resource-manager
worker threads, but custom low-level callbacks must be written lock-free.

**Objects cannot move.** Because structures are transparent and
self-referential once initialized, copying an `ma_device` or `ma_sound` —
e.g. by storing it in a resizing `std::vector` — silently corrupts state.
Heap-allocate or reserve up front. This is the most common C++ user error.

**0.x versioning is real.** v0.10.0 and v0.11.0 each shipped long lists of
breaking API changes[^3][^7], and the maintainer explicitly reserves the
right to break ABI between bug-fix releases. Pin an exact version and vendor
the source; treat upgrades as a migration task, not a bump.

**Compile time.** The implementation TU is large (~4 MB of source). Keep
`miniaudio.c` in its own translation unit so incremental builds don't pay
for it repeatedly.

**Web is the roughest platform.** The default Emscripten path uses the
deprecated ScriptProcessorNode. AudioWorklet support exists behind
`MA_ENABLE_AUDIO_WORKLETS` (v0.11.18+, needs `-sAUDIO_WORKLET=1
-sWASM_WORKERS=1 -sASYNCIFY`)[^8], but the v0.11.22 changelog notes the
worklet path was broken at that release[^5] — verify against your Emscripten
version before shipping.

**Format gaps.** Encoding is WAV-only; Vorbis/Opus decoding is not built in
and requires the split `extras/decoders` wrappers around libvorbis/libopus
or a custom `ma_data_source`[^5]. There is no MIDI support.

**Bus factor.** Development, review, and releases run through one person.
Release cadence has stayed steady for a decade, but a gap of nearly a year
between 0.11.21 and 0.11.22 shows what solo maintenance looks like[^7].

## When to Use / When Not

**Use when:**
- You're shipping a game or app that needs playback, capture, mixing, and
  spatialization without a dependency tree — one .c/.h pair covers desktop,
  mobile, and web.
- You want low-level device access with the option to grow into the engine
  API later without switching libraries.
- License friction matters: public domain / MIT-0 means no attribution
  requirements anywhere in the stack.

**Avoid when:**
- You need a stable ABI or a system-packaged shared library — miniaudio is
  designed to be vendored, not installed.
- Your primary target is the browser and glitch-free worklet audio is a hard
  requirement today.
- You need MIDI, Opus/Vorbis out of the box, or non-WAV encoding.
- You require the OpenAL API surface or middleware-grade tooling (FMOD/Wwise
  territory: authoring tools, banks, profilers).

## Alternatives

- PortAudio/portaudio — device I/O only, ancient and conservative; use it
  when you want a distro-packaged, stable-ABI callback API and nothing else.
- libsdl-org/SDL — use when you already depend on SDL for windowing/input;
  SDL3's audio streams cover playback/capture with format conversion.
- kcat/openal-soft — use for the OpenAL API standard and mature 3D
  positional audio, especially when porting existing OpenAL code.
- jarikomppa/soloud — C++ game audio engine with similar high-level scope
  (mixing, faders, filters); use if you prefer C++ idioms over C structs.
- thestk/rtaudio — C++ realtime device I/O with a long academic lineage; use
  for audio research/tools where decoding and mixing are out of scope.

## History

| Version | Date | Notes |
|---------|------|-------|
| mini_al | 2016-10 | Repository created as `mini_al`, low-level device abstraction. |
| 0.9 | 2019-03-06 | Renamed to miniaudio[^2]. |
| 0.10.0 | 2020-03-07 | Data-conversion API rework, allocation callbacks, period-based latency config[^7]. |
| 0.11.0 | 2021-12-18 | High-level engine, node graph, resource manager merged into core[^3]. |
| 0.11.18 | 2023-08-07 | Opt-in Emscripten AudioWorklet support[^8]. |
| 0.11.22 | 2025-02-24 | `miniaudio.c` added; v0.12 split .c/.h model announced[^5]. |
| 0.11.25 | 2026-03-04 | Current release; decoder sync with dr_libs, POSIX real-time thread fallback[^7]. |

## References

[^1]: miniaudio.h v0.11.25 in-file documentation (authoritative manual). https://raw.githubusercontent.com/mackron/miniaudio/master/miniaudio.h
[^2]: CHANGES.md, v0.9 — 2019-03-06. https://github.com/mackron/miniaudio/blob/master/CHANGES.md
[^3]: CHANGES.md, v0.11.0 — 2021-12-18 (engine, node graph, resource manager). https://github.com/mackron/miniaudio/blob/master/CHANGES.md
[^4]: raylib's raudio module is built on miniaudio. https://github.com/raysan5/raylib/blob/master/src/raudio.c
[^5]: CHANGES.md, v0.11.22 — 2025-02-24 (miniaudio.c, v0.12 plan, extras decoders, worklet status). https://github.com/mackron/miniaudio/blob/master/CHANGES.md
[^6]: miniaudio README, "Supported Platforms / Backends". https://github.com/mackron/miniaudio#supported-platforms
[^7]: CHANGES.md, full changelog. https://github.com/mackron/miniaudio/blob/master/CHANGES.md
[^8]: CHANGES.md, v0.11.18 — 2023-08-07 (`MA_ENABLE_AUDIO_WORKLETS`). https://github.com/mackron/miniaudio/blob/master/CHANGES.md

## Tags

c, audio, audio-playback, audio-capture, single-file-library, cross-platform, game-development, dsp, public-domain, low-latency
