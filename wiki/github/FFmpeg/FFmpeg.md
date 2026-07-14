# FFmpeg/FFmpeg

> The de facto multimedia codec and container toolkit — a set of C libraries and CLI tools that decode, encode, mux, filter, and stream nearly every audio/video format in existence.

[GitHub repo](https://github.com/FFmpeg/FFmpeg) ·
[Official website](https://ffmpeg.org/) ·
[License: LGPL-2.1-or-later (core), GPL-2.0-or-later (optional components)](https://github.com/FFmpeg/FFmpeg/blob/master/LICENSE.md)

## Overview

FFmpeg is the multimedia layer almost everything else stands on. Chrome, VLC, mpv, HandBrake, OBS, Blender, Kodi, and most cloud transcoding pipelines either link its libraries or shell out to its tools[^1]. The project began in 2000 under Fabrice Bellard and has been developed by a large volunteer community since; the GitHub repository here is an official mirror of the canonical git server at git.ffmpeg.org, and pull requests are explicitly ignored — patches go to the ffmpeg-devel mailing list[^2].

The codebase is not one program but seven C libraries plus three CLI front-ends. The libraries (`libavcodec`, `libavformat`, `libavutil`, `libavfilter`, `libavdevice`, `libswscale`, `libswresample`) are the reusable surface; the tools (`ffmpeg`, `ffprobe`, `ffplay`) are thin drivers over them. When people say "FFmpeg" they usually mean the `ffmpeg` CLI, but the durable value is `libavcodec` — the single largest open collection of codec implementations anywhere.

The defining tension is licensing and stability. The core is LGPL, but the most-wanted encoders (x264, x265) are GPL, and a few components (fdk-aac) are non-free and non-redistributable. Which flags you pass at `./configure` determines the legal terms of your binary. Layered on top is a security reality: FFmpeg parses adversarial, malformed input by design, so it has a long CVE history and should never be run un-sandboxed on untrusted files[^3].

## Getting Started

```bash
# Prebuilt binaries (most platforms have them)
brew install ffmpeg            # macOS
apt install ffmpeg             # Debian/Ubuntu (LGPL-safe build)
# Static builds with x264/x265: https://ffmpeg.org/download.html
```

```bash
# Transcode to H.264/AAC in an MP4, scale to 720p, faststart for web
ffmpeg -i input.mov \
  -c:v libx264 -crf 23 -preset medium \
  -c:a aac -b:a 128k \
  -vf "scale=-2:720" \
  -movflags +faststart \
  output.mp4

# Inspect a file as JSON (script-friendly)
ffprobe -v error -print_format json -show_format -show_streams input.mov
```

Note the option grammar: flags before an `-i` apply to that input, flags after the last `-i` apply to output. Order is load-bearing, not cosmetic.

## Architecture / How It Works

The transcoding pipeline is a fixed five-stage graph: **demux → decode → filter → encode → mux**. `libavformat` splits a container (MP4, MKV, MPEG-TS) into elementary packets; `libavcodec` decodes packets into raw frames; `libavfilter` transforms frames through a directed filtergraph; `libavcodec` re-encodes; `libavformat` muxes back into a container. `ffprobe` runs only the left half; a straight remux (`-c copy`) skips decode/filter/encode entirely and is near-instant because no pixels are touched.

`libavcodec` is the center of gravity. Each codec registers as an `AVCodec` with decode/encode callbacks; the same API fronts hundreds of built-in decoders and wraps external encoders (libx264, libx265, libvpx, libaom, libsvtav1) and hardware paths (NVENC/NVDEC, VAAPI, Videotoolbox, QSV, AMF). Hardware acceleration is not automatic — you select it explicitly (`-hwaccel cuda`, `-c:v h264_nvenc`), and mixing hardware decode with software filters forces frames back to system memory, often erasing the speedup.

`libavfilter` is a small dataflow language. Filters are nodes connected by named pads; `-vf "scale=1280:720,fps=30"` is a linear chain, while `-filter_complex` builds arbitrary graphs with splits, overlays, and multiple inputs/outputs. The escaping rules are notoriously hostile: commas, colons, semicolons, and brackets are all syntactically significant, so any filter argument containing them (a `drawtext` string, a Windows path) needs layered backslash escaping.

`libavutil` holds shared primitives (pixel formats, `AVFrame`/`AVPacket`, ref-counted buffers, error codes, options via `AVOptions`). `libswscale` does pixel-format conversion and scaling; `libswresample` does the audio equivalent (resampling, channel-layout remapping, sample-format conversion). These two are where color and audio subtly go wrong — chroma subsampling, color range/matrix (`bt709` vs `bt601`), and full-vs-limited range mismatches all live here.

## Production Notes

**Licensing is the first design decision, not a footnote.** A default LGPL build cannot encode H.264/H.265 (x264/x265 are GPL). `--enable-gpl` unlocks them but relicenses your whole binary as GPL. `--enable-nonfree` (fdk-aac, some others) produces a binary you legally cannot redistribute at all. Distro packages are LGPL-only for this reason; if your product ships FFmpeg with x264, you owe GPL compliance. Audit your `configure` flags before shipping.

**The CLI is not a stable API.** Option semantics, defaults, and filter behavior change across releases; scripts pinned to exact stderr output or implicit defaults break on upgrade. For programmatic use, either link the libraries directly or pin an exact FFmpeg version in your container image. Parse `ffprobe -print_format json`, never scrape human-readable stderr.

**ABI breaks on every major version.** `libavcodec`, `libavformat`, etc. bump SONAME each major release, and deprecated APIs are removed on a schedule (typically after a multi-year deprecation window). Applications linking the libraries (mpv, OBS) must track these; distro upgrades that swap FFmpeg majors can break third-party plugins built against the old ABI.

**Security: treat every input as hostile.** FFmpeg's decoders parse untrusted bitstreams and have a substantial CVE record over the years[^3]. Run transcoding of user-uploaded media in a sandbox (seccomp, gVisor, a locked-down container), disable protocols you don't need (`-protocol_whitelist`), and never let user-controlled input reach `-i` with `file,http,...` protocols open by default — `concat`/`subfile`/HTTP protocol tricks have been used for SSRF and local file reads.

**Performance realities.** `-preset` on x264/x265 trades CPU for size at fixed quality (`-crf`); `ultrafast` and `veryslow` differ by an order of magnitude in speed and meaningfully in bitrate. Hardware encoders (NVENC) are far faster but produce larger files at equal visual quality than a slow software preset. `ffmpeg` gained multithreaded transcoding scheduling in 7.0, but a single filtergraph is still often the serialization bottleneck.

## When to Use / When Not

**Use when:**
- You need to decode, encode, remux, or inspect essentially any media format.
- You want batch/server-side transcoding driven by a CLI or the C libraries.
- You are building a media application and want a codec backend (link `libav*`).
- You need format coverage no other single project matches.

**Avoid / add a layer when:**
- You are embedding media playback or capture into an app with a graph-based pipeline and dynamic reconfiguration — GStreamer's element model fits that better than shelling out to a CLI.
- You want consumer-friendly presets and a GUI — use HandBrake (which wraps FFmpeg libraries) rather than composing flags.
- Your problem is specifically DASH/CMAF/fragmented-MP4 packaging — GPAC is more focused there.
- You cannot accept GPL/non-free licensing terms and also need H.264/H.265 encode — you have a licensing problem no build flag solves cleanly.

## Alternatives

- GStreamer/gstreamer — pipeline/element framework; use it instead when you are building an application with live, reconfigurable media graphs rather than invoking a transcoder.
- HandBrake/HandBrake — GUI transcoder built on FFmpeg's libraries; use it when you want presets and a desktop app instead of CLI flags.
- gpac/gpac — MP4/DASH/CMAF-centric toolkit; use it when your core task is streaming packaging and fragmented-MP4 muxing.
- mpv-player/mpv — playback front-end on top of FFmpeg; use it when the goal is playing media, not converting it.
- libav (fork, now inactive) — 2011 governance-split fork; historically relevant but development effectively stopped, so use FFmpeg.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2000 | Project founded by Fabrice Bellard[^1]. |
| — | 2011-03 | Libav fork over a governance dispute; years of parallel development followed[^2]. |
| 1.0 "Angel" | 2012-09 | First official numbered release. |
| 2.0 | 2013-07 | Native AAC decoder work, filtergraph maturation. |
| 3.0 | 2016-02 | Large merge of Libav changes; native AAC encoder promoted out of experimental. |
| 4.0 | 2018-04 | AV1 decode (dav1d/libaom era), bitstream filter API reshaping. |
| 5.0 | 2022-01 | Deprecated-API cleanup, new subtitle/filter work. |
| 6.0 | 2023-02 | Radiance HDR, further hwaccel and filter additions. |
| 7.0 | 2024-04 | Multithreaded `ffmpeg` CLI transcoding, native VVC/H.266 decoder (experimental). |
| 7.1 | 2024-09 | Stabilization and codec/filter additions on the 7.x line. |

## References

[^1]: FFmpeg — "About FFmpeg" and project history. https://ffmpeg.org/about.html
[^2]: FFmpeg README / developer documentation — patches go to ffmpeg-devel; GitHub PRs are ignored. https://github.com/FFmpeg/FFmpeg/blob/master/README.md
[^3]: FFmpeg security page — advisories and CVE history. https://ffmpeg.org/security.html

## Tags

c, multimedia, video, audio, codec, transcoding, ffmpeg, streaming, libavcodec, cli, video-processing, lgpl
