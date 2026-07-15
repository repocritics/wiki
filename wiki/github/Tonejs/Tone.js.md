# Tonejs/Tone.js

> A Web Audio framework for interactive music in the browser — synths, effects, samples, and a DAW-style transport over the native audio graph.

[GitHub repo](https://github.com/Tonejs/Tone.js) ·
[Official website](https://tonejs.github.io) ·
[License: MIT](https://github.com/Tonejs/Tone.js/blob/dev/LICENSE.md)

## Overview

Tone.js sits one layer above the browser's native Web Audio API. Where raw Web Audio gives you an `AudioContext` and a graph of `AudioNode`s wired sample-by-sample, Tone.js adds the vocabulary a musician expects: named synths (`Tone.Synth`, `Tone.FMSynth`, `Tone.AMSynth`), effects (`Tone.Reverb`, `Tone.FeedbackDelay`, `Tone.Distortion`), samplers, and a global `Transport` that behaves like the arrangement view of a digital audio workstation — start, stop, loop, and change tempo on the fly[^1]. Created by Yotam Mann, it has been maintained since 2014 and is the default answer for "how do I make music in JavaScript" for most of the ecosystem.

The framework's defining tension is that it must impose musical, tempo-relative, restartable time on top of Web Audio's clock, which is a monotonic count of seconds that starts at page load and cannot be paused or rewound. Tone.js resolves this with a lookahead scheduler: JavaScript timers (imprecise) decide *what* to schedule, but each event is committed to the Web Audio clock (sample-accurate) slightly ahead of when it should sound[^2]. The consequence surfaces in every callback — the correct event time is passed in as an argument, and using `Date.now()` or the wall clock instead is the single most common source of jittery output.

As of the v14 line the codebase is a full TypeScript rewrite shipped as tree-shakeable ES modules, which replaced the older monolithic UMD build[^3]. It remains a large dependency, and the API surface (dozens of node types, a signal-automation system, and a time-encoding mini-language) is correspondingly wide.

## Getting Started

```bash
npm install tone          # latest stable
npm install tone@next     # 'dev' branch build, published continuously
```

```javascript
import * as Tone from "tone";

// Browsers block audio until a user gesture — start the context in a click.
document.querySelector("button")?.addEventListener("click", async () => {
  await Tone.start();                          // resume the AudioContext

  const synth = new Tone.Synth().toDestination();
  synth.triggerAttackRelease("C4", "8n");      // play middle C for an 8th note
});
```

Scheduling along the transport, with the sample-accurate `time` passed into the callback:

```javascript
const synth = new Tone.FMSynth().toDestination();
new Tone.Loop((time) => {
  synth.triggerAttackRelease("C2", "8n", time); // use `time`, never Tone.now()
}, "4n").start(0);

Tone.getTransport().bpm.value = 120;
Tone.getTransport().start();
```

## Architecture / How It Works

Every Tone object wraps one or more native Web Audio nodes and exposes the same graph model: `connect()` and `toDestination()` build the routing, mirroring `AudioNode.connect()`. Signal processing stays in the native layer (`GainNode`, `WaveShaperNode`, `OscillatorNode`), so the framework's overhead is orchestration, not sample crunching[^1].

- **Signals.** Parameters like `frequency` and `gain` are `Tone.Signal` objects, not plain numbers. They run at audio rate and support automation curves (`rampTo`, `linearRampTo`, `exponentialRampToValueAtTime`) that are sample-accurate. This is the main thing Tone.js buys you over a thin playback wrapper.
- **Transport.** `Tone.getTransport()` is a restartable, loopable, tempo-adjustable timekeeper layered on the fixed AudioContext clock. It implements the "two clocks" lookahead pattern[^2]: a worker/timer fires ahead of time and schedules the actual audio events into the future, so tempo ramps and loops are possible even though the underlying clock cannot be paused.
- **Time encodings.** Methods that take a time accept numbers (seconds) or notation strings — `"4n"` (quarter note), `"8t"` (eighth triplet), `"1m"` (measure), `"+0.5"` (relative to now). These resolve against the current transport tempo.
- **Instruments.** The built-in synths are monophonic (one voice). Polyphony comes from `Tone.PolySynth`, which is handed a monophonic synth class and manages voice allocation.
- **Context shim.** Tone creates and shims the `AudioContext` through standardized-audio-context[^4] for cross-browser consistency; you can read it with `Tone.getContext()` or swap in your own with `Tone.setContext()`.

The v15 line moved the old global singletons (`Tone.Transport`, `Tone.Destination`, `Tone.context`) to getter functions (`Tone.getTransport()`, `Tone.getDestination()`, `Tone.getContext()`); the singletons still exist but the getters are the documented path[^1].

## Production Notes

- **Autoplay policy is the number-one footgun.** No audio plays until `Tone.start()` runs inside a user-gesture handler (click, keydown). Scheduling before the context is running yields silence or misplaced events — with no error. This bites every first integration.
- **Use the callback `time`, not the wall clock.** JavaScript timers are not sample-accurate. Inside `Loop`/`Part`/`Transport.schedule` callbacks, schedule with the injected `time` argument; reading `Tone.now()` or `Date.now()` there reintroduces jitter.
- **Bundle size.** Tone is a large dependency. `import * as Tone from "tone"` defeats tree-shaking and pulls the whole library; import only the classes you use (`import { Synth, Transport } from "tone"`) to let bundlers drop the rest — the ESM rewrite exists specifically to enable this[^3].
- **SSR / Next.js.** The module touches `window`/`AudioContext` at import time in browser builds; importing it on the server throws. Gate it behind dynamic import or an `useEffect`/client-only boundary.
- **Node lifecycle leaks.** Web Audio nodes are not garbage-collected while still connected. Long-running apps that create synths/effects per note or per interaction must call `.dispose()` explicitly, or memory and CPU climb.
- **Latency vs. stability.** `Tone.getContext().lookAhead` trades responsiveness against timing safety. Lower it for tight interactive playing (at the risk of dropouts under load); raise it for rock-solid sequenced playback.
- **iOS Safari.** The AudioContext suspends aggressively (backgrounding, silent switch, interruptions) and may resume at a different sample rate. Re-check context state on focus, and expect to call `start()`/`resume()` again after interruptions.
- **CPU under polyphony.** Each `PolySynth` voice is a full node subgraph. Dense chords or many concurrent instruments saturate the audio thread on mobile quickly; cap voice counts and prefer samples over heavy synthesis where you can.

## When to Use / When Not

**Use when:**
- You need synthesis, effects, and *musical scheduling* (tempo, loops, quantized events) — not just file playback.
- You want sample-accurate parameter automation and a DAW-like transport without hand-rolling a lookahead scheduler.
- You are building a sequencer, instrument, generative-music piece, or interactive audio toy in the browser.

**Avoid when:**
- You only need to play/loop sound files or sprite sheets — a smaller library is a fraction of the bytes.
- Bundle size is critical and you use one feature; raw Web Audio or a targeted node may be leaner.
- You need audio outside the browser/Web Audio environment, or hard-real-time guarantees the Web Audio thread cannot make.

## Alternatives

- goldfire/howler.js — use when you just need to load, play, loop, and sprite audio files; no synthesis, effects, or musical scheduling.
- chrisguttandin/standardized-audio-context — use when you want cross-browser Web Audio primitives directly, without a framework layer (Tone.js depends on this internally).
- elemaudio/elementary — use when you prefer a functional/declarative DSP model and want to target native as well as the browser.
- tidalcycles/strudel — use when your interest is live-coded, pattern-based algorithmic music rather than an imperative node graph.
- Raw Web Audio API — use when you need minimal footprint and total control and are willing to build scheduling yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014 | First public release by Yotam Mann; monolithic UMD build[^1]. |
| 13.x | — | Final major line before the TypeScript rewrite. |
| 14.0 | — | Full TypeScript rewrite; tree-shakeable ES modules replace the UMD monolith[^3]. |
| 15.x | current | Global singletons superseded by `getTransport()` / `getContext()` / `getDestination()` getters[^1]. |

## References

[^1]: Tone.js README and API documentation. https://tonejs.github.io/docs/
[^2]: Chris Wilson, "A Tale of Two Clocks — Scheduling Web Audio with Precision" — the lookahead scheduling pattern Tone's transport implements. https://web.dev/articles/audio-scheduling
[^3]: Tone.js is distributed as tree-shakeable ES modules following its TypeScript rewrite (v14 line); npm package `tone`. https://www.npmjs.com/package/tone
[^4]: standardized-audio-context — cross-browser Web Audio shim used by Tone.js. https://github.com/chrisguttandin/standardized-audio-context

## Tags

typescript, javascript, web-audio, audio, music, synthesis, sound, scheduling, browser, dsp, sequencer, samples
