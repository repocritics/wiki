# ytdl-org/youtube-dl

The original Python video-downloader for YouTube and 1,000+ other sites. Pre-existed yt-dlp; effectively superseded by it for active use, but remains the historical canonical reference.

## What it is

A Python command-line tool that downloads audio/video from YouTube, Vimeo, and many other sites. Originally authored by Ricardo Garcia in 2010, became the most-used video downloader in the open web throughout the 2010s, and triggered legal action (including the 2020 RIAA DMCA takedown that GitHub later restored). Development has slowed as the maintainer pool thinned; the spiritual successor `yt-dlp` is now the actively-maintained fork that most users should use.

## Key features

- Extractor architecture covering 1,000+ video sites.
- Format selection (best, worst, specific codec, bitrate constraints).
- Subtitle download + embedding.
- Playlist + channel batch download.
- pip-installable; single-binary releases via PyInstaller.
- Unlicense — public-domain-equivalent.

## Tech stack

- Python primary.
- Extractor plugin system — per-site Python modules under `youtube_dl/extractor/`.
- Distributed via PyPI, single-binary releases, and OS package managers.

## When to reach for it

- You specifically need youtube-dl rather than the yt-dlp fork (rare — usually for legacy script compatibility).
- You're studying the project's legal history (the 2020 RIAA takedown + EFF reversal is a landmark OSS-vs-IP case).
- You're maintaining a tool that calls `youtube-dl` by name and want to understand the upstream slowdown.

## When *not* to reach for it

- You're doing active video downloads — `yt-dlp/yt-dlp` is faster, more current with site fixes, and broadly the right default.
- You need YouTube changes from 2023 onward — youtube-dl's extractor maintenance has lagged, and many sites have broken.
- You want a maintained CLI surface — yt-dlp's interface is a superset of youtube-dl's plus active improvement.

## Maturity signal

140k stars, 10k forks, Unlicense, last push February 2026 — slow but not abandoned. 14-year-old project with the most-cited extractor architecture in the video-downloader space. The 4,100 open-issues count reflects accumulated reports the maintainer pool can't address at upstream's slower pace. Anyone doing real-world downloads in 2026 should default to yt-dlp; youtube-dl is the historical reference that yt-dlp forked from.

## Alternatives

- `yt-dlp/yt-dlp` — the actively-maintained fork; the right default for active use.
- `gallery-dl` — use for image-gallery sites.
- Direct `ffmpeg` invocation — use when sources support standard stream protocols and you don't need extractor logic.

## Notes

The 2020 RIAA DMCA takedown of youtube-dl from GitHub, followed by the EFF-led restoration after community + legal pushback, is the defining episode in OSS-vs-DMCA legal history. The Unlicense + the takedown-restoration precedent give youtube-dl unusually clean redistribution rights despite the legally-charged content. For 2026-era practical use, treat this repo as the historical artifact and use yt-dlp.

## Tags

command-line-interface, python, downloader, video, audio, youtube, unlicense, legacy
