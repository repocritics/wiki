# membraneframework/membrane_core

> Multimedia streaming and processing framework for Elixir — media pipelines
> as supervised BEAM processes instead of a shelled-out FFmpeg command.

[GitHub repo](https://github.com/membraneframework/membrane_core) ·
[Official website](https://membrane.stream) ·
[License: Apache-2.0](https://github.com/membraneframework/membrane_core/blob/master/LICENSE)

## Overview

Membrane is a multimedia framework built on Elixir/OTP, developed since 2017
and maintained by Software Mansion[^1]. `membrane_core` is the runtime and
API for media pipelines: ingest/egress over WebRTC, RTSP, RTMP, HLS and HTTP,
transcoding and mixing, and reading/writing containers like MP4, MKV, and
FLV[^1]. The target user is an Elixir team building a long-running, stateful
media server — SFUs, broadcast ingest, recording — not one-shot conversion.

The defining tradeoff: Membrane is an *orchestration* layer, not a codec
implementation. The BEAM provides supervision, isolation, and cheap
concurrency for pipeline topology and stream control; the byte crunching
(decoders, encoders, resamplers) is delegated to C/Rust NIFs via the
companion Unifex/Bundlex tooling. You get Erlang-grade fault recovery for
pipeline logic, but a segfaulting NIF still takes down the whole VM — the
safety story is weakest exactly where media code is riskiest.

At ~1.5k stars it is niche in absolute numbers, but it is the dominant media
framework in the Elixir ecosystem with no serious in-ecosystem competitor,
and actively maintained (v1.3.4 June 2026, pushes in July 2026)[^2]. It ships
as dozens of small Hex packages (`membrane_X_plugin`, `membrane_X_format`);
`membrane_core` is the only mandatory dependency[^1].

## Getting Started

Install: add `{:membrane_core, "~> 1.3"}` to `mix.exs` deps, run
`mix deps.get`. Minimal pipeline — stream an MP3 over HTTP to the sound
card, runnable in `iex` or Livebook[^1]:

```elixir
Mix.install([:membrane_hackney_plugin, :membrane_mp3_mad_plugin,
             :membrane_portaudio_plugin])

defmodule MyPipeline do
  use Membrane.Pipeline

  @impl true
  def handle_init(_ctx, mp3_url) do
    spec =
      child(%Membrane.Hackney.Source{location: mp3_url,
                                     hackney_opts: [follow_redirect: true]})
      |> child(Membrane.MP3.MAD.Decoder)
      |> child(Membrane.PortAudio.Sink)

    {[spec: spec], %{}}
  end
end

Membrane.Pipeline.start_link(MyPipeline, "https://example.com/sample.mp3")
```

## Architecture / How It Works

The core abstractions:

- **Element** — the unit of processing: source, filter, sink, or endpoint.
  Each is its own BEAM process implementing callback behaviours
  (`handle_init`, `handle_setup`, `handle_buffer`, ...) and exchanging
  **buffers** (payload plus pts/dts) and in-band **events** (end-of-stream).
- **Pads** — typed input/output ports declaring accepted stream formats;
  negotiation happens at link time, so a mismatched link fails at startup
  rather than producing garbage output.
- **Pipeline** — the top-level process spawning elements from a declarative
  spec (the `child(...) |> child(...)` DSL) and supervising them as an OTP
  tree; crash groups isolate failing streams in multi-stream servers.
- **Bin** — a reusable sub-graph exposing its own pads, the composition
  mechanism (e.g. an "HLS sink" bin hiding muxer + segmenter + storage).

Flow control is per-pad: automatic demand (default), manual demand, or
push[^3]. Demand propagates upstream so a slow sink throttles the source —
GStreamer's pull model reified as message passing between processes.

Native integration is the coupling story: plugins wrapping libavcodec,
libopus, portaudio etc. build C/Rust via Bundlex and generate NIF bindings
via Unifex, both Membrane sub-projects. This is also why native code is
unsupported on Windows — the project recommends WSL[^4].

## Production Notes

- **Pre-1.0 churn is over, but the ecosystem lags.** Core broke APIs
  frequently until v1.0 (2023-10)[^2]; a core bump can still strand you
  waiting on releases of the separately-versioned plugins — audit your tree
  before upgrading.
- **NIF risk concentrates where the work is.** Codec plugins run native code
  in-process: a codec crash is a VM crash — supervision does not save you —
  and long-running NIF calls can block schedulers, causing VM-wide latency.
- **Flow-control misconfiguration is the classic footgun.** Push-mode pads
  with a slow consumer grow mailboxes without bound; buggy manual demand
  stalls the pipeline silently. Prefer auto demand unless measured otherwise.
- **Soft real time only.** Per-process GC keeps pauses small but the BEAM
  makes no hard-realtime guarantees; low-latency audio timing lives in the
  native sink (e.g. portaudio), not in Elixir.
- **Throughput ceiling.** Every buffer hop between elements is an
  inter-process message — measurable for high-bitrate video with many filter
  stages. Heavy per-frame work belongs inside one native element.
- **Ships an AI-assistant skill.** A `SKILL.md` installable as a Claude Code
  plugin or Cursor rule — Membrane's callback API is under-represented in
  model training data[^5].

## When to Use / When Not

**Use when:**
- You are on Elixir/Phoenix and need media ingest, transcoding, or recording
  inside the same runtime and supervision tree.
- You are building a long-lived media server (SFU, broadcast gateway) with
  dynamically connecting streams and a need for isolated partial failure.
- You need custom logic mid-stream — writing your own filter element is a
  first-class, testable operation.

**Avoid when:**
- You need one-shot file transcoding — plain FFmpeg is simpler and faster.
- Your team has no Elixir experience; the OTP learning curve compounds the
  multimedia one.
- You need maximum codec coverage or squeeze-every-cycle performance —
  GStreamer is ahead there.
- You deploy on native Windows[^4].

## Alternatives

- GStreamer/gstreamer — use when you need the largest codec catalog and an
  in-process C data path, outside the BEAM.
- FFmpeg/FFmpeg — use for batch transcoding with no long-running server.
- bluenviron/mediamtx — use when you want a ready-made RTSP/WebRTC/HLS server
  binary, not a framework to build one.
- pion/webrtc — use when the problem is specifically WebRTC in a Go stack.
- elixir-webrtc/ex_webrtc — use when you need only WebRTC in Elixir without
  adopting the full pipeline framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.1 | 2018-08 | First tagged release[^2]. |
| 0.11.0 | 2022-11 | Spec/API rework toward the `child/2` DSL[^2]. |
| 1.0.0 | 2023-10-16 | API stabilization after ~5 years of 0.x churn[^2]. |
| 1.1.0–1.2.0 | 2024-06 – 2025-02 | Incremental 1.x releases[^2]. |
| 1.3.x | 2026 | Active series (1.3.4, 2026-06); AI-assistant SKILL.md[^2][^5]. |

## References

[^1]: membrane_core README. https://github.com/membraneframework/membrane_core/blob/master/README.md
[^2]: membrane_core releases. https://github.com/membraneframework/membrane_core/releases
[^3]: Membrane API docs (pads, demands, flow control). https://hexdocs.pm/membrane_core/
[^4]: GitHub discussion #857, "Direct support for Windows". https://github.com/orgs/membraneframework/discussions/857
[^5]: Membrane skill for AI assistants (SKILL.md). https://github.com/membraneframework/membrane_core/tree/master/skills/membrane-framework

## Tags

elixir, multimedia, streaming, media-pipeline, webrtc, rtsp, hls, transcoding, otp, framework, audio, video
