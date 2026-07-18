# savonet/liquidsoap

> A statically typed functional scripting language whose programs are radio stations — the de facto open-source engine for netradio and webtv automation.

[GitHub repo](https://github.com/savonet/liquidsoap) ·
[Official website](http://liquidsoap.info) ·
[License: GPL-2.0](https://github.com/savonet/liquidsoap/blob/main/COPYING)

## Overview

Liquidsoap is a media stream automation tool from the Savonet project, in continuous development since 2003[^1]. Instead of a config file, you write a program: the scripting language is statically typed with full type inference, functional, and equipped with domain operators — playlists, live input takeover, crossfades, silence detection, muxing, encoding — that compose into a running station. It is written in OCaml and is, practically speaking, the only serious open-source option in its niche: AzuraCast and LibreTime, the two popular web-UI radio suites, both drive Liquidsoap under the hood.

The 1,695 stars and 158 forks understate real deployment for exactly that reason — most operators consume it indirectly through those suites or the official Docker images. Development is active (pushed within days as of mid-2026) but concentrated in essentially two core developers (Romain Beauxis, Samuel Mimram), which shows up as a real bus-factor and support-bandwidth constraint: only the current release cycle receives bugfixes[^2].

The defining tradeoff is language-versus-config. A bespoke typed functional language buys expressivity no radio config format can match (per-show jingle logic, custom fade curves, HTTP-triggered switches), at the cost of a genuine learning curve and a maintainer-acknowledged reality that even minor releases can change script behavior — the README itself strongly recommends running a staging environment before any upgrade[^2].

## Getting Started

```bash
opam install liquidsoap        # via OCaml's package manager (OCaml >= 4.14)
# or without a toolchain:
docker run -v "$(pwd)":/app savonet/liquidsoap:v2.4.5 /app/radio.liq
```

```liquidsoap
# radio.liq — playlist radio with live DJ takeover
music = playlist("/srv/music")
live  = input.harbor("live", port=8005, password="hackme")

# Live input preempts the playlist; switch immediately, not at track end
radio = fallback(track_sensitive=false, [live, music])

# Outputs must be infallible: pad any gap with silence
radio = mksafe(radio)

output.icecast(%mp3(bitrate=128),
  host="localhost", port=8000, password="hackme",
  mount="radio.mp3", radio)
```

Debian/Ubuntu `.deb` packages and Alpine `.apk` packages are published per release; the Debian packages require the third-party deb-multimedia.org repository for codec dependencies[^2].

## Architecture / How It Works

**Language layer.** The `liquidsoap-lang` component is a general-purpose functional language: type inference, labeled and optional arguments, row-polymorphic records/methods, pattern matching. Much of the "standard library" (operators like `crossfade`, playlist handling) is itself written in Liquidsoap under `src/libs/`. Tooling exists (prettier formatter, tree-sitter grammar, VS Code extension), which is more than most embedded DSLs get.

**Source graph.** A script builds a pull-based graph of *sources*. At startup the type checker and a fallibility analysis run over the whole graph: an output must be fed by an *infallible* source (one that can always produce data) or explicitly declare `fallible=true`. `mksafe` / `fallback` convert fallible sources into infallible ones. This static check — refusing to start a station that could go silent — is the design's signature move, and also the first error every newcomer hits.

**Clocks.** Every source is attached to a clock (wallclock CPU time, a soundcard's clock, or a decoder's timestamps). Mixing sources from different clocks requires an explicit `buffer`, and clock mismatch errors are a classic intermediate-level footgun. Version 2.3.0 re-implemented the entire streaming cycle and clock system: frames are now assembled from immutable content chunks carrying PTS/DTS, with track marks and metadata as first-class content — a rewrite explicitly aimed at enabling OCaml multicore use in future releases and at retiring 20-year-old design decisions[^3].

**I/O and encoding.** `input.harbor` embeds an Icecast-compatible ingest server so DJs can connect live sources directly to the script. Outputs cover Icecast, HLS, SRT, files, and soundcards. Since 2.0, FFmpeg is deeply integrated — `%ffmpeg` encoders/decoders and inline FFmpeg filters — including stream *copy*, i.e. relaying or repackaging encoded content (e.g. Icecast in, HLS out) without re-encoding[^4]. FFmpeg support tracks the last two major versions (currently 7 and 8)[^2]. A telnet/HTTP command server allows runtime control (skip, push requests, metadata).

## Production Notes

- **Upgrades are the main operational risk.** The maintainers state that minor versions "may be incompatible" and that even bugfix releases can break scripts that relied on incorrect operator behavior; they strongly recommend a staging environment[^2]. Treat Liquidsoap upgrades like application deployments, not package updates.
- **Forced upgrade cadence.** Only the current release cycle is supported: 2.4.x is supported, 2.3.x already is not[^2]. There is no LTS. Combined with the point above, expect to budget script-migration work roughly yearly (migration notes are published per release).
- **Major rewrites land in dot releases.** 2.3.0 replaced the core streaming engine[^3]; early 2.3.x releases saw regressions before stabilizing at 2.3.3. By contrast the project advertised 2.4.0 as containing "no major rewrite of the core components"[^5] — read release notes to judge risk.
- **CPU model.** One long-running process does everything in real time; each encoded output costs an encoder's worth of CPU continuously. FFmpeg stream copy is the escape hatch for relay/repackaging workloads. The streaming loop is not yet multicore-parallel; 2.3's engine rewrite is groundwork, not the payoff[^3].
- **Startup-time failures.** Fallibility errors ("that source is fallible") and clock errors ("a source cannot belong to two clocks") abort at startup. Irritating at first, but they are the mechanism that keeps a mis-wired station from silently dying at 3 a.m.
- **Install surface.** Feature availability depends on which optional OCaml libraries were compiled in; the official Docker images are the lowest-pain path and what AzuraCast effectively standardizes on.

## When to Use / When Not

**Use when:**
- You run an internet radio or webtv station with nontrivial logic: live DJ handoff, scheduled shows, jingle insertion, crossfades, silence detection.
- You need HLS packaging or multi-format simulcast from one live source.
- You are building a streaming platform and want a scriptable, headless engine rather than a monolithic app.
- AzuraCast/LibreTime's UI ceiling is too low and you need the layer beneath.

**Avoid when:**
- You only need to distribute already-encoded streams — Icecast alone does that; Liquidsoap is the source-side automation, not the fan-out server.
- You want a web UI and no code — AzuraCast or LibreTime give you Liquidsoap without writing Liquidsoap.
- Your job is one-shot transcoding or a simple push pipeline — the ffmpeg CLI is simpler and has no long-running language to learn.
- You need to embed a media pipeline inside an application — GStreamer is a library; Liquidsoap is a standalone daemon with its own language.

## Alternatives

- azuracast/azuracast — turnkey web-UI radio management built on Liquidsoap; use it when you want a product, not a language.
- libretime/libretime — calendar/scheduling-first radio automation (Airtime fork), also driving Liquidsoap underneath.
- xiph/Icecast-Server — the distribution server Liquidsoap typically feeds; sufficient alone when sources are already encoded and automated.
- FFmpeg/FFmpeg — use directly for one-shot transcodes or simple relay pipelines without station logic.
- gstreamer/gstreamer — use when embedding media processing into your own application rather than operating a standalone streaming daemon.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2003–2010 | Savonet project era (SourceForge); language and operator model take shape[^1]. |
| 1.0 | 2011 | First stable release; repository moved to GitHub 2011-12. |
| 1.4.0 | 2019-10-01 | Last 1.x feature line. |
| 2.0.0 | 2021-10-04 | ~2-year rewrite: full FFmpeg integration, video via ffmpeg, stream copy[^4]. |
| 2.1.0 | 2022-07-17 | Rolling-release model adopted around this cycle[^6]. |
| 2.2.0 | 2023-07-22 | Track muxing/demuxing (multitrack), HLS and sound-processing improvements[^6]. |
| 2.3.0 | 2024-11-26 | Streaming cycle and clocks re-implemented; immutable content chunks; multicore groundwork[^3]. |
| 2.4.0 | 2025-09-01 | Language ergonomics (argument destructuring, `null` handling); no core rewrite[^5]. |
| 2.4.5 | 2026-06-15 | Current supported stable; 2.5.x in development on `main`[^2]. |

## References

[^1]: Liquidsoap README ("Copyright 2003-2026 Savonet team"). https://github.com/savonet/liquidsoap#readme
[^2]: Liquidsoap README — "Release Details": versioning caveats, staging recommendation, support matrix, FFmpeg policy. https://github.com/savonet/liquidsoap#release-details
[^3]: Liquidsoap 2.3.0 release notes — 2024-11-26. https://github.com/savonet/liquidsoap/releases/tag/v2.3.0
[^4]: Liquidsoap 2.0.0 release notes — 2021-10-04. https://github.com/savonet/liquidsoap/releases/tag/v2.0.0
[^5]: Liquidsoap 2.4.0 release notes. https://github.com/savonet/liquidsoap/releases/tag/v2.4.0
[^6]: Liquidsoap 2.2.0 blog post — 2023-07-22. https://www.liquidsoap.info/blog/2023-07-22-liquidsoap-2.2.0/

## Tags

ocaml, streaming, internet-radio, audio, video, hls, icecast, ffmpeg, dsl, media-automation
