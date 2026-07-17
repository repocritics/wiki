# RustAudio/cpal

> Low-level cross-platform audio I/O in Rust — one enumerate/open/stream API over WASAPI, CoreAudio, ALSA, AAudio, and the Web Audio API.

[GitHub repo](https://github.com/RustAudio/cpal) ·
[License: Apache-2.0](https://github.com/RustAudio/cpal/blob/master/LICENSE)

## Overview

cpal (Cross-Platform Audio Library) is the base audio I/O layer of the RustAudio
ecosystem. It does exactly one thing: enumerate audio hosts and devices, negotiate
a stream configuration (sample rate, channel count, sample format, buffer size),
and hand you a real-time callback that either fills an output buffer or drains an
input one. It does not mix, resample, decode files, apply effects, or manage a
graph — those belong to higher layers. If you want to "just play an MP3", the same
org's `rodio` sits on top of cpal and is the intended entry point[^1]. cpal is what
you reach for when you are building that higher layer yourself, or when you need
direct control of the device and the callback thread.

The defining tension is that cpal is a *thin* abstraction over deeply *unlike*
platform backends. WASAPI, CoreAudio, ALSA, AAudio, JACK, PipeWire, PulseAudio and
the Web Audio API disagree about buffer models, threading, default-device
semantics, sample formats, and error reporting. cpal presents one Host → Device →
Stream shape across all of them, but the abstraction is intentionally leaky: buffer
sizes, latency, real-time scheduling, and which errors are recoverable all remain
platform-specific, and the caller is expected to know that. This is honest for a
low-level library, but it means "works on my machine" and "works cross-platform"
are genuinely different amounts of work here.

cpal is old by Rust-crate standards (the repository dates to 2014[^2], predating
Rust 1.0) and is still actively developed, with commits and backend work landing
regularly. It is a foundational dependency — bevy_audio, kira, and rodio all route
through it — so its API is conservative and breaking changes are infrequent but
real across the pre-1.0 `0.x` series.

## Getting Started

```toml
# Cargo.toml
[dependencies]
cpal = "0.15"
```

On Linux/BSD you also need the ALSA development headers even if you plan to use
JACK, PipeWire, or PulseAudio: `libasound2-dev` (Debian/Ubuntu) or `alsa-lib-devel`
(Fedora).

```rust
use cpal::traits::{DeviceTrait, HostTrait, StreamTrait};

fn main() -> anyhow::Result<()> {
    let host = cpal::default_host();
    let device = host.default_output_device()
        .expect("no output device available");
    let config = device.default_output_config()?;

    let stream = device.build_output_stream(
        &config.config(),
        move |data: &mut [f32], _: &cpal::OutputCallbackInfo| {
            for sample in data.iter_mut() {
                *sample = 0.0; // fill with silence; real code writes audio here
            }
        },
        move |err| eprintln!("stream error: {err}"),
        None, // Option<Duration> timeout
    )?;

    stream.play()?;
    std::thread::sleep(std::time::Duration::from_secs(1));
    Ok(())
}
```

The callback runs on a high-priority OS audio thread. Everything inside it must be
real-time safe: no allocation, no locks, no blocking I/O. Communicate with it via a
lock-free ring buffer (e.g. `ringbuf`) or atomics.

## Architecture / How It Works

The model is three traits: **Host** (a backend API such as WASAPI or ALSA),
**Device** (a soundcard or endpoint under that host), and **Stream** (an open,
running callback). `default_host()` picks the platform default; `host_from_id()`
selects an optional backend (ASIO, JACK, PipeWire, PulseAudio) when its Cargo
feature is enabled. Enabling a backend is a compile-time choice via features —
there is no runtime plugin loading.

Sample format is handled two ways. `build_output_stream::<T>` fixes the format at
compile time (`f32`, `i16`, etc.); `build_output_stream_raw` takes a
`SampleFormat` value and hands you an untyped `Data` buffer for runtime dispatch.
Most code uses the typed path and calls `default_output_config()` to discover what
the device actually supports rather than assuming.

The callback is the whole point and the whole hazard. cpal does not own a mixer or
a clock loop; each Stream is a thin wrapper around the backend's own callback
mechanism, invoked by an OS-owned real-time thread. Historically (before the ~0.11
redesign) cpal used a single `EventLoop::run` that multiplexed all streams through
one callback; the current per-stream closure API replaced that and is what all
modern code targets[^2]. `BufferSize::Default` defers to the system default, which
on ALSA can be anything from a PipeWire quantum (~1024 frames) up to `u32::MAX` on
misconfigured hardware; `BufferSize::Fixed(n)` requests a specific size but is not
honored by every backend.

On Linux the backend landscape is the messiest part. ALSA is the always-required
base; JACK, PipeWire, and PulseAudio are opt-in features layered on top. When
PipeWire or PulseAudio is running it holds the ALSA `default` device exclusively,
so a second stream opened through the plain ALSA host fails with `DeviceBusy` — the
documented fix is to open the `pipewire`/`pulse` bridge devices or, better, use the
native `pipewire`/`pulseaudio` features[^3]. The `realtime` / `realtime-dbus`
features promote the callback thread to `SCHED_FIFO`; `realtime-dbus` goes through
`rtkit` so you avoid hand-editing `limits.conf`.

## Production Notes

- **The callback is real-time; treat it like an interrupt handler.** Any
  allocation, mutex, `println!`, or syscall in it risks an xrun (buffer
  under/overflow) heard as a click or dropout. This is the single most common cpal
  bug in the wild and cpal cannot protect you from it — the API hands you a
  `&mut [f32]` and trusts you.
- **Default buffer size is not a latency guarantee.** A deep `BufferSize::Default`
  on ALSA/PipeWire can make output appear to "fast-forward" because samples are
  consumed far faster than they play. Query `default_output_config()?.buffer_size()`
  for the valid range and request a `Fixed` size when latency matters.
- **Errors arrive on two channels.** Setup errors return from `build_*_stream`;
  runtime errors (device unplugged, RT promotion denied, backend hiccup) arrive on
  the error callback, sometimes after the stream has been running. Robust apps must
  handle mid-stream device loss, not just startup failure.
- **Real-time priority silently degrades.** `RealtimeDenied` is reported only when
  promotion is attempted and fails (missing `rtprio` limit); the stream keeps
  playing at normal priority with higher glitch risk. On Linux this usually means
  the user is not in the `audio` group or `limits.conf` is unset.
- **ASIO on Windows is a build project, not a dependency.** The `asio` feature
  needs the Steinberg ASIO SDK, LLVM/Clang (for `bindgen`), `LIBCLANG_PATH`, and a
  Visual Studio toolchain; cross-compiling adds MinGW-w64 include-path juggling.
  Budget real setup time before promising low-latency Windows audio.
- **WASM needs extra flags.** The default `wasm-bindgen` (Web Audio API) path is
  fine, but the lower-latency `audioworklet` backend requires nightly,
  `-Zbuild-std` with atomics, and cross-origin isolation headers for
  `SharedArrayBuffer`.
- **MSRV and OS minimums vary by backend.** Most backends target Rust 1.85, but
  PulseAudio needs 1.88 and tvOS/audioworklet need nightly; macOS CoreAudio
  requires 14.2 (loopback capture 14.6+). A CI matrix that only tests one platform
  will miss these.
- **Pre-1.0 API churn.** cpal is still `0.x`; minor bumps (e.g. into the 0.13→0.15
  range) have moved the stream-builder signatures. Pin the version and read the
  changelog before upgrading, especially if you depend on it transitively via
  rodio or bevy.

## When to Use / When Not

**Use when:**
- You are building your own mixer, synth, DAW, game audio engine, or streaming
  layer and need direct device + callback control.
- You want one codebase to reach WASAPI, CoreAudio, ALSA, AAudio, and the browser
  without hand-writing each backend.
- You need input capture, output playback, or duplex with explicit format and
  buffer negotiation.

**Avoid when:**
- You just want to play or decode audio files — use `rodio` (which wraps cpal) and
  skip the real-time callback entirely.
- You need built-in mixing, effects, spatialization, or an audio graph — that is
  `kira` or an engine's audio module, not cpal.
- You cannot invest in per-platform testing; the abstraction is thin and the hard
  parts (latency, RT scheduling, device quirks) stay platform-specific.

## Alternatives

- RustAudio/rodio — higher-level playback/decoding built on cpal; use when you want to play sounds, not manage streams.
- tesselode/kira — game/app audio engine with mixing, tweens, and effects on top of cpal; use when you need a mixer, not raw I/O.
- mackron/miniaudio — single-file C low-level audio library with similar scope and its own Rust bindings; use when you want a C dependency or a backend cpal lacks.
- andrewrk/libsoundio — mature C cross-platform low-level library; use when you need a battle-tested C core callable from multiple languages.
- PortAudio/portaudio — long-established C audio I/O library; use when you need decades of platform coverage and existing PortAudio expertise.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-12 | Repository created under RustAudio, pre-Rust-1.0[^2]. |
| ~0.11 | ~2019 | API redesign from single `EventLoop::run` to per-stream callback closures (Host/Device/Stream traits)[^2]. |
| 0.15.x | 2023–2026 | Current line; ongoing backend work (PipeWire, AAudio, WebAssembly audioworklet, realtime scheduling)[^3]. |

## References

[^1]: cpal README — "For higher-level audio playback and capture, consider Rodio." https://github.com/RustAudio/cpal
[^2]: RustAudio/cpal repository (created 2014-12-11) and API documentation. https://docs.rs/cpal
[^3]: cpal README — supported platforms, optional backend features, ALSA/PipeWire/PulseAudio and real-time notes. https://github.com/RustAudio/cpal

## Tags

rust, audio, audio-io, cross-platform, low-level, dsp, real-time, wasapi, coreaudio, alsa, webassembly, sound
