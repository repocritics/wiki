# jellyfin/jellyfin

> A self-hosted media server that catalogs, transcodes, and streams your own media library — the fully-free fork of Emby.

[GitHub repo](https://github.com/jellyfin/jellyfin) ·
[Official website](https://jellyfin.org) ·
[License: GPL-2.0](https://github.com/jellyfin/jellyfin/blob/master/LICENSE)

## Overview

Jellyfin is a media server: it scans directories of movies, TV, music, and
photos, enriches them with metadata from online providers, and streams them to
client apps (web, Android, iOS, Roku, Kodi, tvOS, and others) over an HTTP API.
It occupies the same product category as Plex and Emby — a private "Netflix for
your own files" — with one deliberate difference: everything is free software
under GPL-2.0, with no premium tier, paywalled hardware transcoding, or account
gating[^1].

The project is a hard fork of Emby's 3.5.2 release, created in December 2018
after Emby's codebase went closed-source[^1]. That origin explains most of the
codebase's character: it inherited a large, mature, but proprietary-era .NET
application and has been community-modernizing it ever since. Much of the tree
still carries `Emby.*` and `MediaBrowser.*` namespaces from that lineage, sitting
alongside newer `Jellyfin.*` projects. The defining tension of the repo is this
slow, volunteer-driven cleanup of inherited code — not a green-field design.

This repository is the **server backend and API only**. The browser UI lives in
a separate repo (jellyfin-web), the patched transcoder in jellyfin-ffmpeg, and
each client platform in its own repo under the `jellyfin` org. With ~54k stars
and commits landing daily, it is one of the most active self-hosting projects on
GitHub, though development is steady-state maintenance and incremental features
rather than rapid reinvention.

## Getting Started

Docker is the most common deployment. Hardware transcoding requires passing the
GPU device through to the container.

```bash
docker run -d --name jellyfin \
  -v /path/to/config:/config \
  -v /path/to/cache:/cache \
  -v /path/to/media:/media:ro \
  -p 8096:8096 \
  --device /dev/dri:/dev/dri \        # Intel/AMD VAAPI/QSV passthrough (optional)
  jellyfin/jellyfin
```

Then open `http://<host>:8096` and run the setup wizard to add libraries and a
user. Building the server from source (for contributors) needs the .NET 10 SDK
and a copy of the web client[^2]:

```bash
git clone https://github.com/jellyfin/jellyfin.git
cd jellyfin
dotnet run --project Jellyfin.Server \
  --webdir /absolute/path/to/jellyfin-web/dist
```

## Architecture / How It Works

Jellyfin runs as an ASP.NET Core host. A single process serves the REST API
(documented via Swagger at `/api-docs/swagger`), optionally hosts the static web
client files, and coordinates background tasks. Key subsystems:

- **Library scanner** — walks configured folders, matches files to items, and
  resolves metadata. Metadata comes from pluggable providers (TheMovieDB, TVDB,
  MusicBrainz, and others) or from local `.nfo`/artwork files. Results are
  persisted to a SQLite database in the config directory.
- **Media encoding** — playback decisions are made per client per stream:
  *direct play* (client streams the original file untouched), *direct stream*
  (remux container only), or *transcode* (re-encode video/audio). Transcoding
  shells out to a bundled jellyfin-ffmpeg subprocess. Anything the client can't
  natively decode — an unsupported codec, a subtitle format that must be burned
  in, an HDR→SDR tone-map — forces a transcode.
- **Plugins** — loaded as .NET assemblies at startup; they extend metadata
  providers, add channels, or hook scheduled tasks.
- **Persistence** — historically a hand-rolled SQLite layer inherited from Emby;
  the project has been migrating persistence toward Entity Framework Core over
  recent releases, one entity area at a time.

The coupling story is the honest part. The server, the web client, ffmpeg, and
each client app version independently, and the HTTP API is the contract between
them. Because the API grew out of Emby's, some endpoints and data shapes are
legacy and awkward. The core playback/transcoding logic is intricate and
under-documented — it is where most subtle client-compatibility bugs live.

## Production Notes

**Transcoding is the whole game.** A server that only ever direct-plays needs
almost no CPU; one transcoding several 4K streams will saturate a machine.
Hardware acceleration (Intel QSV/VAAPI, NVIDIA NVENC, AMD) cuts this
dramatically but is the single most fragile part of a deployment: it depends on
host drivers, correct `/dev/dri` passthrough in Docker, and matching
jellyfin-ffmpeg builds. HDR tone-mapping and subtitle burn-in are especially
expensive and often the reason a "why is my CPU pinned" thread starts.

**Keep config off network filesystems.** The SQLite library database does not
tolerate NFS/SMB well; running `/config` on a network share invites database
locks and corruption. Put config on local disk and mount only the media
read-only over the network.

**Remote access is your responsibility.** There is no built-in TLS or hardened
public-exposure story. The community norm is to front Jellyfin with a reverse
proxy (nginx, Traefik, Caddy) and, ideally, keep it behind a VPN rather than
exposing port 8096 to the internet directly.

**Client fragmentation.** "Jellyfin plays this file" is client-specific. The web
player, official Android/iOS apps, Kodi via the jellyfin plugin, and third-party
clients (Findroid, Streamyfin, Infuse) each have different codec support and
quirks. Test the file with the clients you actually use before assuming a format
works.

**Upgrades** are generally in-place — database migrations run automatically on
startup — but take a backup of `/config` first, since a failed migration on a
large library is painful to unwind. Pin an image tag rather than tracking
`latest` if you want to control when a major version lands.

**Plugin ecosystem** is smaller and less curated than Plex's; useful plugins
exist but some are abandoned, and there is no strong review gate.

## When to Use / When Not

**Use when:**
- You want a fully free, no-paywall media server you fully control.
- You object to telemetry, account requirements, or per-feature licensing.
- You self-host already and can manage a reverse proxy and (optionally) GPU
  drivers for transcoding.
- Your clients mostly direct-play, or you have hardware to transcode.

**Avoid when:**
- You want a turnkey, minimal-maintenance experience with the most polished
  mobile/TV apps — Plex is more consistent out of the box.
- You need seamless out-of-home streaming with zero networking setup — Plex's
  relay/discovery handles this; Jellyfin expects you to configure access.
- You only have music, or only photos — a single-purpose server is lighter.
- You can't provide any transcode-capable hardware and your clients need it.

## Alternatives

- Plex — closed-source commercial ancestor's rival; more polished clients and
  easier remote streaming, but freemium (Plex Pass) and account-gated. Use when
  you value UX polish over software freedom.
- emby/embyserver — the proprietary project Jellyfin forked from; similar
  feature set with a premium tier. Use if you specifically want Emby's paid
  features and don't mind closed source.
- xbmc/xbmc (Kodi) — a local-first media center, not a client-server streaming
  system. Use when playback happens on the same box as the library.
- navidrome/navidrome — music-only, Subsonic-API-compatible, far lighter. Use
  when you only need audio streaming.
- photoprism/photoprism — photo/video library with AI tagging. Use when your
  media is a photo collection, not movies and TV.

## History

| Version | Date | Notes |
|---------|------|-------|
| 10.0.0 | 2018-12 | First release; hard fork of Emby 3.5.2, ported to cross-platform .NET[^1]. |
| 10.7.0 | 2021-03 | API/plugin modernization, quicksync and tone-mapping improvements. |
| 10.8.0 | 2022-06 | Trickplay/transcoding work, large scanner and metadata changes. |
| 10.9.0 | 2024-03 | Continued EF Core migration, ffmpeg and hardware-accel updates. |
| 10.10.0 | 2024-12 | Trickplay images, further persistence and API refactors. |

## References

[^1]: Jellyfin project — about / origin as a fork of Emby 3.5.2 (Dec 2018).
  https://jellyfin.org/docs/general/about
[^2]: Jellyfin server README — build-from-source and prerequisites (.NET SDK,
  jellyfin-ffmpeg, separate web client). https://github.com/jellyfin/jellyfin

## Tags

media-server, self-hosted, csharp, dotnet, transcoding, streaming, ffmpeg, emby-fork, gpl, homelab, video
