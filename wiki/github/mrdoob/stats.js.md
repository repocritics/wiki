# mrdoob/stats.js

> A ~1 KB in-page widget that graphs frames-per-second, frame time, and JS heap size for the current tab.

[GitHub repo](https://github.com/mrdoob/stats.js) ·
[Official website](http://mrdoob.github.io/stats.js/) ·
[License: MIT](https://github.com/mrdoob/stats.js/blob/master/LICENSE)

## Overview

stats.js is a tiny performance HUD by Ricardo Cabello (mrdoob), the author of
three.js. It draws a small canvas panel in the corner of a page showing one of
three metrics at a time: **FPS** (frames rendered in the last second), **MS**
(milliseconds spent between a `begin()`/`end()` pair), or **MB** (allocated JS
heap). It was written to debug WebGL/canvas render loops and remains the default
"is my animation dropping frames" instrument across the three.js ecosystem and
much of the creative-coding world[^1].

Its defining characteristic is scope discipline. It does not profile function
calls, attribute cost, sample the GPU, or persist history — it draws one number
and a scrolling sparkline. This is why it is one file, has no dependencies, and
has barely changed in years: it is close to "done." The flip side is that it
measures wall-clock frame time from JS, so it cannot tell you *why* a frame is
slow, only that it is, and its memory panel depends on a non-standard Chrome API.

The project is stable rather than actively developed. At ~9.1k stars and ~1.2k
forks[^2] it is widely depended upon, but commits are sparse and the last push
to `master` was in late 2024[^2] — treat it as a finished utility, not a moving
target.

## Getting Started

```bash
npm install stats.js
```

```javascript
import Stats from 'stats.js';

const stats = new Stats();
stats.showPanel(0);            // 0: fps, 1: ms, 2: mb, 3+: custom
document.body.appendChild(stats.dom);

function animate() {
  stats.begin();

  // ---- the code you want measured ----
  render();
  // -------------------------------------

  stats.end();
  requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

`begin()`/`end()` bracket the work you care about, so the MS panel reports the
cost of *your* frame rather than idle time. The older combined API — a single
`stats.update()` call per frame — still exists and measures wall-clock time
between successive calls; the README bookmarklet uses it[^1].

## Architecture / How It Works

The whole library is a couple hundred lines. `new Stats()` returns an object
exposing `dom` (a positioned `<div>`), `showPanel(id)`, `begin()`, `end()`,
`update()`, and `addPanel(panel)`. Each metric is a `Stats.Panel` — a small
`<canvas>` (drawn at device-pixel scale, roughly 80×48 logical px) that renders a
label, the current value, min/max seen, and a right-scrolling bar graph. Only the
panel selected by `showPanel` is visible; clicking the widget cycles panels.

The measurement loop is deliberately naive:

- **MS** is `performance.now()` at `end()` minus `performance.now()` at `begin()`.
- **FPS** counts how many `end()` calls happen inside each one-second window and
  reports that count when the window rolls over.
- **MB** reads `performance.memory.usedJSHeapSize` / `jsHeapSizeLimit`. This is a
  non-standard, Chromium-only property; in other browsers the MB panel is simply
  never added[^1]. For meaningful numbers Chrome must be started with
  `--enable-precise-memory-info`, otherwise the value is bucketed/quantized.

`addPanel` lets you register custom panels (`new Stats.Panel(name, fg, bg)`) and
feed them values via `panel.update(value, maxValue)`, which is how people bolt on
draw-call counts, triangle counts, or GPU timings from elsewhere. stats.js itself
does none of that instrumentation — it only owns the drawing and the FPS/MS/MB
math.

Distribution is a single ES module plus a UMD/minified `build/stats.min.js`. It
has zero runtime dependencies and no build-time coupling to three.js despite the
shared author; three.js examples merely bundle a copy.

## Production Notes

- **Development tool, not a metrics pipeline.** stats.js is meant to sit on
  screen while you watch it. It keeps no history you can export, emits no events,
  and has no programmatic accessor for the last value beyond reading the panel's
  own state. For dashboards or RUM telemetry, sample `performance.now()` /
  `PerformanceObserver` yourself.
- **The MS panel measures only what you bracket.** Anything outside
  `begin()`/`end()` — browser layout, paint, compositing, other rAF callbacks —
  is invisible. A "16 ms" reading with a stuttering page usually means the cost is
  in code you didn't wrap.
- **FPS is capped by the display and rAF.** On a 60 Hz panel the ceiling is 60;
  on 120 Hz ProMotion it is 120. A steady 60 is not proof of headroom — check MS,
  where a low number (e.g. 4 ms) reveals spare budget.
- **MB is Chrome-only and coarse.** Without `--enable-precise-memory-info` the
  heap figure is rounded for fingerprinting protection, and the panel does not
  appear at all in Firefox/Safari. Do not gate logic on it.
- **The widget itself costs a little.** Drawing a canvas panel every frame is
  cheap but non-zero; it competes for the same main thread you are measuring.
  Remove it from production builds rather than hiding it with CSS.
- **DPI/positioning.** `stats.dom` is absolutely positioned top-left with a fixed
  z-index. On high-DPI or transformed layouts you often need to restyle
  `stats.dom` (position, scale) to keep it visible and legible.
- **No GPU visibility.** stats.js cannot see GPU-bound stalls; a render loop can
  report a fast MS while the GPU is the true bottleneck. Use stats-gl or Spector
  when the cost is on the GPU side.

## When to Use / When Not

**Use when:**
- You want an instant, dependency-free FPS/frame-time readout on a canvas, WebGL,
  or requestAnimationFrame render loop.
- You're already in the three.js / creative-coding world and want the conventional
  overlay.
- You need a lightweight custom counter (draw calls, entity count) via `addPanel`.

**Avoid when:**
- You need to know *where* time goes — use the browser's profiler/Performance panel.
- You need GPU timings — use stats-gl or Spector.js.
- You need exportable, historical, or server-collected metrics — build on
  `PerformanceObserver`/RUM instead.
- You're on Firefox/Safari and specifically want the memory panel.

## Alternatives

- RenaudRohlinger/stats-gl — modern drop-in successor with GPU timing (WebGL2/
  WebGPU) and CPU/GPU split; use it when you need GPU-side numbers, not just JS.
- spite/rStats — multi-metric, higher-density graphs with arbitrary user counters;
  use when one number at a time is too coarse.
- BabylonJS/Spector.js — full WebGL frame capture and command inspection; use to
  debug *why* a frame is expensive, not just how long it took.
- Chrome DevTools Performance panel — flamecharts, attribution, layout/paint
  timing; use when you need a real profiler rather than an in-page HUD.
- pmndrs/stats.js wrappers (e.g. r3f `<Stats />`) — use when you're in React
  Three Fiber and want the same widget wired into the render loop for you.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010-04-08 | First public commit; debugging aid alongside three.js[^2]. |
| panel-API rewrite | ~2016 | `begin()`/`end()`, `showPanel`, `addPanel`, `Stats.Panel` replace the old single `update()` model[^1]. |
| 0.17.0 (r17) | ~2020 | Latest npm release; API effectively frozen since[^3]. |
| maintenance | 2024-10-11 | Most recent commit to `master`; no new release[^2]. |

## References

[^1]: stats.js README — usage, panels, bookmarklet, and the MB/`--enable-precise-memory-info` caveat. https://github.com/mrdoob/stats.js/blob/master/README.md
[^2]: GitHub repository metadata (stars, forks, created 2010-04-08, last push 2024-10-11), retrieved 2026-07-17. https://github.com/mrdoob/stats.js
[^3]: stats.js on npm — package `stats.js`, latest published `0.17.0`. https://www.npmjs.com/package/stats.js

## Tags

javascript, performance-monitoring, fps, profiling, webgl, three-js, browser, canvas, dev-tools, frontend
