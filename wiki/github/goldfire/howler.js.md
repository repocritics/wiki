# goldfire/howler.js

> Cross-browser JavaScript audio library that defaults to the Web Audio API and falls back to HTML5 `<audio>`, exposing one API for both.

[GitHub repo](https://github.com/goldfire/howler.js) ·
[Official website](https://howlerjs.com) ·
[License: MIT](https://github.com/goldfire/howler.js/blob/master/LICENSE)

## Overview

howler.js is a browser audio library maintained primarily by James Simpson (GoldFire Studios), first published in 2013[^1]. Its reason to exist is that web audio has two incompatible substrates: the Web Audio API (low-latency, buffer-based, good for games and effects) and the HTML5 `<audio>` element (streaming, good for long files, universally available). howler.js presents a single `Howl` object that uses Web Audio by default and transparently falls back to HTML5 Audio when Web Audio is unavailable or when you opt in for streaming — so calling code does not branch on which backend is active.

The target users are game developers and interactive-media authors who need reliable sound effects, sprites, looping, fading, and 3D/stereo spatial audio without hand-writing `AudioContext` plumbing and the many mobile/browser workarounds it requires. The library is dependency-free, ships as roughly 7 KB gzipped for the core, and splits into a `howler.core` build plus an optional `howler.spatial` plugin[^2].

The defining tension is the two-backend abstraction itself: it is what makes howler.js useful, and also the source of most surprises. Web Audio and HTML5 Audio differ in latency, seeking behavior, memory model, codec support, and mobile-unlock rules, and the unified API cannot fully hide those differences. Choosing `html5: true` versus the default silently changes the performance and feature profile of a sound.

## Getting Started

```bash
npm install howler
# or: yarn add howler
```

```javascript
import { Howl, Howler } from 'howler';

const sound = new Howl({
  src: ['sfx.webm', 'sfx.mp3'],   // ordered by preference; first playable wins
  volume: 0.5,
  onend: () => console.log('finished'),
});

const id = sound.play();          // returns a numeric sound id
sound.fade(0.5, 0.0, 1000, id);   // fade this instance out over 1s
Howler.volume(0.8);               // global master volume
```

Provide multiple `src` entries in codec-preference order; howler picks the first the browser can play. Extensionless URLs or base64 data URIs require an explicit `format` array. For large or live audio, set `html5: true` so playback can start before the whole file downloads and decodes.

## Architecture / How It Works

A `Howl` is a group definition (a source plus options); each call to `.play()` spawns a `Sound` instance identified by a numeric id, and most methods accept that id to target one instance or omit it to affect the whole group. Instances are recycled from a per-`Howl` pool (`pool`, default 5) to avoid reallocating nodes; ids drained from the pool become dead references that silently no-op.

Under Web Audio, sources are fetched via XHR, decoded once into an `AudioBuffer`, and cached; playback wires `AudioBufferSourceNode` → `GainNode` → destination, with the spatial plugin inserting a `PannerNode`/`StereoPannerNode`. Under HTML5 mode each sound drives an `<audio>` element instead; these cannot be spatialized with the full Web Audio panner and behave differently on seek. The two paths share the same public methods but not the same guarantees.

The `Howler` global owns cross-cutting state: the shared `AudioContext`, the master gain node, codec detection (`Howler.codecs(ext)`), and the mobile-unlock machinery. Because HTML5 `<audio>` elements must each be unlocked by a user gesture on mobile, howler maintains a global pool of pre-unlocked nodes (`html5PoolSize`, default 10) created on first interaction and shared across all `Howl` instances. Web Audio similarly requires the `AudioContext` to be resumed inside a user gesture; `autoUnlock` handles this on the first touch/click, firing the `unlock` event.

The distribution is deliberately modular: `howler.core` covers only Web-Audio/HTML5 parity; `howler.spatial` is an add-on requiring core, providing `stereo()`, `pos()`, `orientation()`, and `pannerAttr()`. The default `howler` bundle includes both.

## Production Notes

- **Mobile autoplay is a hard gate.** Audio will not start until a real user gesture occurs. `autoUnlock` covers the common case, but the *first* play must originate from within a touch/click handler; deferring it (promises, timeouts) can leave audio locked. Test on physical iOS Safari, not just desktop.
- **`autoSuspend` can bite the first play after idle.** The `AudioContext` suspends after 30 s of inactivity to save power and resumes on next playback, but the resume adds latency and has historically caused a glitch or delay on the first sound after a quiet period. For latency-sensitive games, set `Howler.autoSuspend = false`.
- **Web Audio keeps decoded PCM in memory.** Decoded buffers are uncompressed and cached for the life of the `Howl`; many or large sounds can consume far more RAM than the compressed file sizes suggest. Call `.unload()` (or `Howler.unload()`) to release buffers in long-running apps; leaks here are a common cause of tab memory growth.
- **`html5: true` changes the feature set.** It enables streaming and lowers memory use, but full 3D spatial audio requires Web Audio; HTML5 seeking is less precise, and you can exhaust `html5PoolSize` if you play many concurrent streaming sounds. Raise the pool size or reuse instances.
- **Codec fallbacks are your responsibility.** Safari historically lacks Ogg/Opus; always ship an MP3 or AAC fallback alongside WebM/Ogg and list sources in preference order. `Howler.codecs('opus')` lets you probe support at runtime.
- **TypeScript types are external.** The library is plain JavaScript; types come from the community `@types/howler` package and can lag API additions.
- **Maintenance is low-tempo and single-maintainer.** The project is mature and widely embedded, but the open-issue backlog is large and releases are infrequent[^3]; treat it as a stable, mostly-finished dependency rather than one under active feature development. Pin versions and expect to work around long-standing edge cases yourself.

## When to Use / When Not

**Use when:**
- You need reliable sound effects, sprites, looping, and fading in a game or interactive app across desktop and mobile browsers.
- You want Web Audio's latency where available but a graceful HTML5 fallback and streaming for long files, behind one API.
- You want 3D/stereo spatial audio without writing `PannerNode` graphs by hand.
- You want a small, zero-dependency drop-in rather than a full audio framework.

**Avoid when:**
- You need synthesis, effects graphs, precise musical scheduling, or a transport clock — that is Tone.js territory, not a playback library.
- You need fine-grained DSP (custom `AudioWorklet` nodes, analysers, filters) — use the Web Audio API directly or a thinner wrapper.
- You only ever play one simple background track — a plain `<audio>` element is smaller and sufficient.
- You require an actively iterating dependency with fast issue turnaround.

## Alternatives

- Tonejs/Tone.js — use instead when you need synthesis, effects, and sample-accurate musical scheduling rather than clip playback.
- pixijs/sound — use instead when your project is already built on PixiJS and you want audio integrated with its loader and ticker.
- chrisguttandin/standardized-audio-context — use instead when you want raw Web Audio but need consistent cross-browser behavior without howler's abstraction.
- CreateJS/SoundJS — comparable multi-fallback audio library; historically similar goals but far less maintained today.
- Native Web Audio API — use directly when you need full control, minimal footprint, and are willing to handle mobile unlock and codec fallbacks yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013 | Initial release: Web Audio with HTML5 Audio fallback[^1]. |
| 2.0 | 2017 | Major rewrite: per-instance sound ids, sound pool, sprites, and the split core/spatial plugin architecture[^2]. |
| 2.1 | 2018 | Refinements to mobile unlock and HTML5 pooling. |
| 2.2 | 2020 | `autoUnlock`, `html5PoolSize`, and further mobile/codec fixes; current 2.2.x line. |

## References

[^1]: howler.js project, GoldFire Studios — repository created 2013-01-28. https://github.com/goldfire/howler.js
[^2]: howler.js README, "Included distribution files" and Plugin: Spatial (core vs spatial builds). https://github.com/goldfire/howler.js#documentation
[^3]: howler.js issue tracker (open-issue backlog and release cadence). https://github.com/goldfire/howler.js/issues

## Tags

javascript, audio, web-audio-api, html5-audio, game-audio, spatial-audio, browser, sound, cross-browser, library, mit-license
