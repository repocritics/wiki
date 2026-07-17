# RustAudio/rodio

> Pure-Rust audio playback built as a tree of lazy sample iterators on top of cpal.

[GitHub repo](https://github.com/RustAudio/rodio) ·
[Documentation](https://docs.rs/rodio) ·
[License: MIT OR Apache-2.0](https://github.com/RustAudio/rodio/blob/master/LICENSE-APACHE)

## Overview

Rodio is the default high-level audio playback library for Rust. It sits one layer
above [cpal](https://github.com/RustAudio/cpal), the RustAudio project's
cross-platform device I/O crate: cpal opens the output device and delivers the
real-time callback, rodio provides decoding, mixing, per-source controls
(volume, speed, pause, seek), and effect chains. It has been part of the
RustAudio org since 2015[^1] and is the crate most Rust game and app projects
reach for first when they need "play this sound."

The defining design choice is that **audio is modeled as `Iterator`**. A `Source`
is an iterator of samples plus a little metadata (channel count, sample rate,
optional total duration). Every effect — `amplify`, `fade_in`, `low_pass`,
`speed`, `repeat_infinite`, `take_duration` — is a combinator that wraps one
source in another, exactly like `map`/`filter` on a normal iterator[^2]. Nothing
is precomputed; samples are pulled on demand from the audio callback thread. This
makes composition cheap and idiomatic, but it also means the cost of decoding and
every effect in the chain is paid on the real-time thread, which is the root of
most of rodio's production footguns.

Rodio is a 0.x crate and has stayed there. The API is reworked meaningfully
across minor versions — the maintainers ship a dedicated `UPGRADE.md` for each
breaking release[^3] — so pinning a version and reading the upgrade guide before
bumping is part of normal use, not an exception.

## Getting Started

```toml
[dependencies]
rodio = "0.22"
```

```rust
use std::io::BufReader;
use std::fs::File;
use rodio::{Decoder, OutputStreamBuilder, Sink};

fn main() {
    // The stream handle owns the device. If it is dropped, audio stops.
    let stream = OutputStreamBuilder::open_default_stream().unwrap();
    let sink = Sink::connect_new(stream.mixer());

    let file = BufReader::new(File::open("sound.ogg").unwrap());
    let source = Decoder::new(file).unwrap();
    sink.append(source);

    sink.sleep_until_end(); // block until the queue drains
}
```

The exact constructors (`OutputStreamBuilder`, `Sink::connect_new`) are recent;
older tutorials use `OutputStream::try_default()` and `Sink::try_new`, which were
renamed/reshaped in the 0.20–0.21 series. Check the version you actually pinned.

## Architecture / How It Works

Three concepts do all the work:

- **`Source`** — `Iterator<Item = Sample>` plus `channels()`, `sample_rate()`,
  `current_span_len()`, and `total_duration()`. Decoders, sine generators, and
  every effect implement it. Samples are `f32` in current rodio.
- **`Sink`** — an append-only queue of sources played back to back, with control
  handles (volume, speed, pause, stop, seek) implemented as atomics read on the
  audio thread. Dropping a `Sink` detaches it; `detach()` lets a sound outlive its
  handle.
- **Mixer** — sums the `f32` outputs of every active source. Sources whose sample
  rate or channel count differ from the output are wrapped in a converter that
  **resamples by linear interpolation** and up/down-mixes channels.

Playback flow: cpal opens the device and repeatedly calls rodio's callback asking
for the next block of samples. The mixer walks its active sources, calls
`next()` on each, sums them, and writes the block. Because this is a pull model,
the entire source chain — file decode, resample, every effect combinator — runs
*inside the real-time callback*. There is no decode-ahead buffer unless you add
one (`buffered()`).

Decoding is delegated. Since the 0.17–0.20 era the default decoder is
[Symphonia](https://github.com/pdeljanov/Symphonia), a pure-Rust demux/decode
crate covering MP3, AAC, FLAC, WAV, OGG/Vorbis and more[^1]. Format-specific
pure-Rust backends (claxon for FLAC, lewton for Vorbis, hound for WAV, minimp3
for MP3) remain available behind feature flags for smaller builds. Which formats
compile in is controlled entirely by Cargo features.

On Linux, cpal selects a host in the order **PipeWire → PulseAudio → ALSA**,
falling back when a backend is not compiled in or not present at runtime; ALSA is
always the base layer, so `libasound2-dev` is a hard build requirement even when
PulseAudio is the runtime path[^1].

## Production Notes

- **Keep the stream alive.** The single most common bug: `let (_stream, handle) =
  ...` and then letting `_stream` drop at the end of a scope. When the stream
  object is dropped the device closes and playback stops silently. The stream must
  live as long as you want any sound to play.
- **Decoding runs on the audio thread.** Because sources are pulled from the
  real-time callback, an expensive decode or a slow effect can miss the deadline
  and produce clicks/underruns. For large or compressed files, decode once and
  wrap in `buffered()`, or decode to memory ahead of time. Never do file I/O or
  allocation-heavy work lazily inside a custom `Source::next()`.
- **Resampling is linear interpolation.** Fine for games and UI sounds; audible
  for high-fidelity music resampling. If you need quality sample-rate conversion,
  resample upstream with a dedicated crate rather than relying on rodio's implicit
  conversion.
- **Breaking changes across minor versions.** 0.x releases regularly rename types
  and restructure the output-stream/sink API (`OutputStreamBuilder`, span vs.
  frame terminology, sample-type unification to `f32`). Read `UPGRADE.md` before
  every bump; do not assume examples from an older version compile[^3].
- **Symphonia inflates build size and compile time.** The default decoder pulls in
  broad format support. For embedded or size-sensitive builds, disable default
  features and enable only the format-specific decoders you need — or build with
  no cpal dependency at all for decode-only (`into_file`) use.
- **Latency is cpal's buffer, not rodio's.** Default buffer sizes come from the
  device/host; low-latency use requires configuring cpal's stream, not rodio.
- **MSRV** is a rolling policy of at least 6 months behind stable[^1]; the target
  must have 32-bit `f32` hardware and ≥32-bit atomics to keep up in real time.

## When to Use / When Not

**Use when:**
- You want simple "load a file and play it" or a mixer of sound effects in a Rust
  app or game without learning audio DSP.
- You need cross-platform playback (Windows/macOS/Linux/Android/iOS via cpal) from
  one API.
- You want to compose effects declaratively as iterator combinators.

**Avoid when:**
- You need low-latency, sample-accurate scheduling, tweens, or a game-audio engine
  with mixer buses — reach for a purpose-built engine (kira) on top of cpal.
- You need audio capture/recording or full-duplex — that is cpal's domain, not
  rodio's.
- You need professional resampling quality or a plugin/DSP graph — use dedicated
  DSP crates.
- You cannot tolerate periodic API churn across minor versions.

## Alternatives

- RustAudio/cpal — the layer rodio is built on; use it directly when you need
  device enumeration, capture, or manual real-time callbacks.
- pdeljanov/Symphonia — use it when you only need to decode/demux audio and are
  not playing it back through a device.
- tesselode/kira — use it when you want a game-oriented audio engine with tweens,
  clocks, mixer tracks, and sub-sample scheduling instead of a plain sink.
- mackron/miniaudio — use it (via bindings) when you want a single-file C audio
  library with capture and its own decoders, outside the Rust ecosystem.
- bevy engine's `bevy_audio` — use it when you are already in Bevy; it wraps rodio
  itself, so choosing it is choosing rodio with an ECS front end.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015 | Initial release under the RustAudio org, built on cpal[^1]. |
| 0.11 | 2020 | Long-standing pre-Symphonia line with per-format decoders. |
| ~0.17 | 2023 | Symphonia adopted as the default decoding backend[^1]. |
| 0.19 | 2024 | Effect/source chain and API cleanups. |
| 0.20 | 2025 | Output-stream/sink API rework; migration guide shipped[^3]. |
| 0.21 | 2025 | Further breaking API changes; dedicated `UPGRADE.md`[^3]. |
| 0.22 | 2026 | Current line; `f32` sample unification, builder-based stream API. |

Exact minor-version dates are approximate; treat crates.io and the changelog as
authoritative before citing a specific release date.

## References

[^1]: RustAudio/rodio README and repository metadata (cpal playback, Symphonia
default decoder, per-format decoder features, Linux host order, MSRV policy).
https://github.com/RustAudio/rodio
[^2]: rodio `Source` trait and combinator documentation. https://docs.rs/rodio/latest/rodio/source/trait.Source.html
[^3]: rodio upgrade guide (`UPGRADE.md`). https://github.com/RustAudio/rodio/blob/master/UPGRADE.md

## Tags

rust, audio, audio-playback, sound, cpal, symphonia, game-audio, dsp, cross-platform, iterator-api
