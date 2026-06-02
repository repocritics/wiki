# iptv-org/iptv

A community-curated collection of publicly-available IPTV channel streams from around the world, distributed as M3U playlists.

## What it is

A repository of publicly-broadcasted television streams (live TV channels from many countries) curated into M3U playlist files. Users point an IPTV player (VLC, Kodi, mpv, ffplay, IPTV-specific apps) at the playlist URL and get live access to thousands of channels. Streams come from public broadcast sources only — paid / region-locked / DRM-protected streams are explicitly out of scope. Sister repos under the `iptv-org` org handle EPG (electronic program guide), API, and tooling.

## Key features

- Thousands of channels across many countries grouped by region, language, category.
- Multiple playlist variants — `index.m3u` (everything), per-country, per-language, per-category playlists.
- Automated dead-stream pruning via scheduled CI.
- Companion EPG (TV guide), API, and tooling repos under the same org.
- Unlicense — public-domain-equivalent.
- Static site at iptv-org.github.io for playlist browsing.

## Tech stack

- TypeScript primary (curation tooling, CI scripts).
- M3U / M3U8 as the canonical playlist format.
- GitHub Actions for automated stream-health checks.

## When to reach for it

- You're configuring a personal IPTV setup and want a curated playlist source.
- You're building an IPTV-aware app (player, EPG aggregator, regional bouquet generator) and need a clean source.
- You're studying community-maintained large-scale media curation as a model.

## When *not* to reach for it

- You want guaranteed legality across all jurisdictions — public-broadcast streams have varying redistribution rules per country; verify local laws.
- You want premium / region-locked / DRM streams — explicitly out of scope.
- You want SLAs — streams come from public sources with no uptime guarantees; the curation tooling prunes dead ones, but quality varies.

## Maturity signal

116k stars, 6.2k forks, Unlicense, last push 2026-06-02 (the day this page was generated). 7-year-old project with active CI-driven stream-health automation. Open-issues count of 67 is unusually low — the maintainers process per-stream reports continuously. Unlicense + organizational structure (multiple coordinated repos) signal long-term institutional intent.

## Alternatives

- Per-country government broadcaster playlists (BBC iPlayer, NHK, etc.) — use when you need licensed, region-locked, official streams.
- Commercial IPTV providers — use when you want premium content with SLAs.
- Plex / Jellyfin live-TV add-ons — use when you have a tuner card and want to manage local broadcast.

## Notes

Stream legality is jurisdiction-specific; the project itself only catalogs publicly-broadcast streams, but users in some regions may face local restrictions on redistribution. Unlicense is the most permissive available license — vendoring playlists into derivative tools is friction-free. The org-wide split (curation here, tooling elsewhere, API elsewhere) is a deliberate architecture for sustainability.

## Tags

awesome-list, iptv, m3u, television, playlist, typescript, unlicense, curated-directory, media
