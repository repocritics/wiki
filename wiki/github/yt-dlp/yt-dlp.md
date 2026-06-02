# yt-dlp/yt-dlp

A feature-rich, actively-maintained fork of youtube-dl — the canonical command-line audio/video downloader for the open web.

## What it is

A Python CLI that downloads audio/video from YouTube and 1,500+ other sites. Originally forked from the upstream youtube-dl project to address its slower update cadence; has since become the de facto downloader as upstream youtube-dl development slowed. Features SponsorBlock integration to skip sponsor segments, extensive output-format control, plugin / extractor architecture, and an active extractor team that ships fixes for site changes within days. Distributed under the Unlicense (effectively public domain).

## Key features

- 1,500+ supported sites — YouTube, Vimeo, Twitch, Twitter/X, TikTok, archive.org, BBC iPlayer, news outlets, and more.
- SponsorBlock integration to skip sponsor segments, intros, and outros in downloads.
- Fine-grained format control — pick best audio + best video, force a specific codec, embed subtitles + chapters + thumbnails.
- Extractor plugin architecture — adding a new site is a discrete Python extractor file.
- pip-installable, single-binary releases, Docker images, Homebrew formulae.
- Configuration file + named "extractor args" for site-specific options.
- Unlicense — public-domain-equivalent, frictionless redistribution.

## Tech stack

- Python primary; runs on Python 3.9+.
- Single-binary release built with PyInstaller for users who don't want a Python install.
- Plugin / extractor system implemented as Python modules under `yt_dlp/extractor/`.

## When to reach for it

- You're archiving content you've created or licensed, downloading podcasts, or pulling videos for offline viewing.
- You're building an audio/video pipeline that needs source-of-truth metadata from a site (titles, descriptions, chapters).
- You need a single tool that handles 1,500+ sites with a stable API surface.

## When *not* to reach for it

- You're downloading copyrighted material you don't have rights to. yt-dlp is a tool; the legal responsibility is yours.
- You need a GUI — yt-dlp is CLI-only; the GUI front-ends (e.g. youtube-dl-gui forks) are separate projects.
- You need real-time streaming rather than downloads.

## Maturity signal

167k stars, 14k forks, Unlicense, last push 2026-05-25 — actively maintained. 6-year-old project with a sustained extractor-maintenance cadence that's faster than the upstream youtube-dl ever managed. Open-issues count of 2,544 tracks per-site breakage reports, which are the dominant maintenance load when you support 1,500 sites. The Discord community + GitHub Releases cadence are both healthy signals.

## Alternatives

- `ytdl-org/youtube-dl` (upstream) — use only if you specifically need the upstream rather than the fork; yt-dlp is generally ahead on fixes.
- `gallery-dl` — use when you're downloading from image-gallery sites (Pixiv, DeviantArt, etc.) rather than video.
- ffmpeg `ffmpeg -i <stream>` directly — use when the source supports raw stream download and you don't need site-specific metadata.

## Notes

Unlicense is the most permissive choice — vendor-friendly, redistribution-friendly, public-domain-equivalent. The extractor architecture is the durability mechanism: when YouTube or Twitch change their JS player, extractor maintainers push a patch within hours, and the install path (pip, brew, scoop) propagates the fix quickly. License + cadence + Discord community make yt-dlp the rare CLI tool where "the maintainers care" is the headline feature.

## Tags

command-line-interface, python, downloader, video, audio, youtube, unlicense, awesome-list, multimedia
