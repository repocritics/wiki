# katspaugh/wavesurfer.js

> Browser waveform renderer and audio player: decodes audio client-side and draws an interactive, seekable waveform.

[GitHub repo](https://github.com/katspaugh/wavesurfer.js) ·
[Official website](https://wavesurfer.xyz) ·
[License: BSD-3-Clause](https://github.com/katspaugh/wavesurfer.js/blob/main/LICENSE)

## Overview

wavesurfer.js is a client-side JavaScript library that renders an audio file as an interactive waveform and wraps playback controls around it. It has existed since 2012[^1], which makes it one of the longest-lived Web Audio visualization projects still under active development — the most recent commit predates this page by days. Its niche is narrow and stable: given a URL or `Blob`, draw the waveform, let the user click to seek, and expose events. It is not a DAW, an effects rack, or a general Web Audio wrapper, and the maintainer is explicit about refusing that scope creep[^2].

The defining architectural fact is that wavesurfer decodes audio *in the browser* via the Web Audio API to compute peaks. This is what makes it zero-backend and easy to drop in, and it is also its central tradeoff: large or long files can exhaust memory and fail to decode. The escape hatch — supplying pre-computed peaks — is a first-class feature, but one many users only discover after hitting the wall.

Version 7 (2023) was a full TypeScript rewrite with a new plugin system and Shadow DOM rendering[^3]. It is not API-compatible with the v6 line, so most "wavesurfer broke on upgrade" reports trace to that boundary. This page describes v7, the current major.

## Getting Started

```bash
npm install --save wavesurfer.js
```

```js
import WaveSurfer from 'wavesurfer.js'

const wavesurfer = WaveSurfer.create({
  container: '#waveform',
  waveColor: '#4F4A85',
  progressColor: '#383351',
  url: '/audio.mp3',
})

wavesurfer.on('ready', (duration) => console.log('ready', duration))
wavesurfer.on('interaction', () => wavesurfer.playPause())
```

Plugins are imported separately from `wavesurfer.js/dist/plugins/`:

```js
import Regions from 'wavesurfer.js/dist/plugins/regions.esm.js'
wavesurfer.registerPlugin(Regions.create())
```

TypeScript types ship with the package; no `@types/wavesurfer.js` is needed. A UMD build is also served from a CDN as a `WaveSurfer` global for script-tag usage.

## Architecture / How It Works

The core splits into a **player** (an HTML `<audio>`/`<video>` element or a Web Audio graph), a **renderer** (canvas drawing inside a Shadow DOM host), and a **decoder** that produces per-channel peak arrays. `WaveSurfer.create()` wires these together and emits a typed event bus (`ready`, `play`, `timeupdate`, `interaction`, `seeking`, `finish`, etc.).

Playback normally rides on a native media element, which means the browser handles buffering and codec support and wavesurfer stays a thin controller. Decoding to peaks, however, uses `AudioContext.decodeAudioData`, which loads and decodes the *entire* file into memory. The waveform draw and the audio playback are therefore two separate paths over the same source — a subtlety that surfaces as visual/audio drift on VBR files, where the decoder's timing assumptions and the media element's real timeline disagree[^2].

v7 renders into a **Shadow DOM** tree to isolate its internal CSS from the host page. Consequences: you cannot style internals with ordinary selectors; you reach them through the `::part()` pseudo-selector against elements carrying a `part` attribute (`::part(cursor)`, `::part(region)`). This is cleaner than v6's global class names but trips up anyone porting old stylesheets.

Plugins are the extension surface. Official ones — Regions, Timeline, Minimap, Envelope, Record, Spectrogram, Hover — are maintained in-repo and versioned with the core, registered via `registerPlugin()`. Each plugin is its own ESM entry point so bundlers can tree-shake unused ones.

## Production Notes

- **Large files fail silently-ish.** Because decoding is all-in-memory, long clips (podcasts, hour-long recordings) can crash the decode. The supported fix is to pre-compute peaks — commonly with BBC's `audiowaveform` CLI — and pass them via the `peaks` option along with an explicit `duration`[^4]. Treat this as the default architecture for anything over a few minutes, not an optimization.
- **Streaming needs pre-decoded peaks.** True streaming playback is only supported when you supply peaks and duration up front; otherwise wavesurfer wants the whole file to decode.
- **CORS is on you.** wavesurfer fetches the audio URL to decode it, so cross-origin audio must send `Access-Control-Allow-Origin`. There is no client-side workaround; a large share of "waveform won't load" issues are CORS, not wavesurfer.
- **VBR mismatch.** Variable-bit-rate MP3s can desync the waveform cursor from the audio. Fixes are re-encoding to CBR or using the Web Audio playback shim, which is more accurate but takes over the playback path.
- **Shadow DOM styling.** Any design system that assumed it could reach into wavesurfer's markup must migrate to `::part()`. Server-side rendering frameworks also need the instance created client-side only, since it touches `AudioContext` and the DOM.
- **v6 → v7 is a rewrite, not an upgrade.** Options, plugin registration, and events all changed. Budget a real migration; pinning to v6 is a legitimate choice for legacy apps, but v6 is no longer where fixes land.

## When to Use / When Not

**Use when:**
- You need an interactive, seekable waveform for short-to-medium audio in the browser with no backend.
- You want regions, a timeline, a spectrogram, or mic recording without assembling them from raw Web Audio.
- You want a mature, single-purpose library with typed events and a stable, narrow scope.

**Avoid when:**
- You need to edit, cut, mix, or apply effects to audio — wavesurfer is a player with a visualization, and the maintainer explicitly declines that scope.
- Your audio is long/large and you cannot pre-compute peaks server-side (memory-bound decode will bite).
- You need multitrack/DAW-style timelines out of the box.
- You only need playback with no visualization — a waveform renderer is dead weight then.

## Alternatives

- bbc/peaks.js — use when your audio is large and you already generate peaks server-side with `audiowaveform`; it is built around pre-computed data.
- naomiaro/waveform-playlist — use when you need a multitrack, DAW-like editor with a timeline rather than a single-track player.
- goldfire/howler.js — use when you want robust cross-browser audio playback (sprites, spatial) and no visual waveform at all.
- Tonejs/Tone.js — use when the job is Web Audio synthesis, scheduling, and effects, not visualizing an existing file.
- CookPete/react-player — use when you mainly need a React media player and the waveform is optional.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012 | First release; canvas waveform over Web Audio[^1]. |
| 2.x–6.x | 2016–2022 | Canvas rendering, MediaElement/WebAudio backends, class-based plugins. |
| 7.0 | 2023 | Full TypeScript rewrite, new plugin system, Shadow DOM rendering, ESM plugin entries[^3]. |

## References

[^1]: wavesurfer.js repository, created 2012-03-04. https://github.com/katspaugh/wavesurfer.js
[^2]: README, "How do I connect wavesurfer.js to Web Audio effects?" — the maintainer states wavesurfer is "just a player with a waveform visualization" and declines cut/effect/processing scope. https://github.com/katspaugh/wavesurfer.js#questions
[^3]: wavesurfer.js v7 documentation and guide. https://wavesurfer.xyz/docs/
[^4]: README FAQ, "Does wavesurfer support large files?" — recommends pre-decoded peaks generated with a tool such as audiowaveform. https://github.com/katspaugh/wavesurfer.js#questions

## Tags

javascript, typescript, audio, waveform, web-audio, audio-player, visualization, frontend, browser, media
