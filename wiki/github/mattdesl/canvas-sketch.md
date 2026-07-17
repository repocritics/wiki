# mattdesl/canvas-sketch

> A JavaScript framework for generative artwork whose defining feature is
> print-quality export — still images, loops, and plotter art rather than
> interactive web pages.

[GitHub repo](https://github.com/mattdesl/canvas-sketch) ·
[Documentation](https://github.com/mattdesl/canvas-sketch/blob/master/docs/README.md) ·
[License: MIT](https://github.com/mattdesl/canvas-sketch/blob/master/LICENSE.md)

## Overview

canvas-sketch is a framework for making generative art in JavaScript and the
browser, written by Matt DeLambo and first published in 2018[^1]. Its own README
still labels it `[beta]` and the package has never left the `0.x` range — which
matters, because the roadmap it advertises (API docs, unit tests, a GUI, a
gallery mode) was largely never delivered[^2]. What did ship well is the part
that distinguishes it from every general creative-coding library: a rendering
and export pipeline built around physical output. You declare a sketch in
real-world units and DPI, and a keystroke writes a correctly-sized PNG to disk.

The audience is artists, plotter/pen-art makers, and creative coders who treat
the browser as a rendering surface, not a deployment target. A canvas-sketch
program is an *authoring tool* for producing artifacts — an archival-print PNG,
an MP4 loop for social, a sequence of SVG paths for an AxiDraw plotter — not a
web application you hand to end users. The defining tension: the runtime is
modern and framework-agnostic (a plain 2D context, WebGL via three.js/regl/ogl,
or SVG interchangeably), while the companion CLI that makes it ergonomic is built
on mattdesl's older browserify/budo tooling. The library is timeless; the tooling
shows its age.

## Getting Started

The runtime library and the CLI are two separate npm packages: `canvas-sketch`
(the framework you `require`) and `canvas-sketch-cli` (the dev server, exporter,
and scaffolder). The `npx` path installs and runs the CLI for you.

```sh
mkdir my-sketches && cd my-sketches
# scaffold a new sketch and open it in the browser
npx canvas-sketch-cli sketch.js --new --open
```

```js
const canvasSketch = require('canvas-sketch');

const settings = {
  dimensions: 'a4',        // preset paper size; also [width, height]
  pixelsPerInch: 300,      // export resolution
  units: 'in'              // width/height are given to you in inches
};

const sketch = () => {
  // return a render function, called for each frame
  return ({ context, width, height }) => {
    context.fillStyle = 'hsl(0, 0%, 98%)';
    context.fillRect(0, 0, width, height);

    const fill = context.createLinearGradient(0, 0, width, height);
    fill.addColorStop(0, 'cyan');
    fill.addColorStop(1, 'orange');
    context.fillStyle = fill;
    const m = 1 / 4; // quarter-inch margin, in real units
    context.fillRect(m, m, width - m * 2, height - m * 2);
  };
};

canvasSketch(sketch, settings);
```

In the browser, `Cmd/Ctrl + S` exports a PNG that measures exactly 21 × 29.7 cm
at 300 DPI — printable on quality paper without rescaling.

## Architecture / How It Works

`canvasSketch(sketch, settings)` returns a *SketchManager*. The manager owns the
canvas element, the animation loop (`requestAnimationFrame`), the device pixel
ratio, resizing, and export. The `sketch` factory runs once and returns a render
function; the render function is invoked every frame with a single `props`
object — `context`, `width`, `height`, `time`, `playhead`, `frame`, `deltaTime`,
`exporting`, and more. Everything the framework knows, it hands you through
`props`; there is no hidden global state.

The `settings` object is the whole configuration surface. `dimensions` accepts
paper presets (`'a4'`, `'letter'`, `'postcard'`) or an explicit `[w, h]` pair;
`units` (`px`/`cm`/`mm`/`in`/`m`) rescales the coordinate space so you draw in
physical units while the manager handles pixel math via `pixelsPerInch`.
Animation is opt-in with `animate: true`, and `duration` + `fps` define a
`playhead` value in `[0, 1]` that loops — the standard way to author seamless
loops. The runtime is renderer-neutral: `context` draws 2D, WebGL templates hand
you a `gl` context deferring to three.js/regl/ogl, and an SVG mode emits vector
output for plotters. The CLI's `--template` flag (`three`, `p5`, `pen`, …)
scaffolds each.

Export is the load-bearing subsystem. In the browser alone, the save keystroke
uses `canvas.toBlob`; the CLI's value-add is a local dev-server endpoint the page
POSTs to, so exports land in a real output *folder* instead of the browser's
download prompt — and for animations it writes a numbered PNG sequence, then
shells out to `ffmpeg` to mux an MP4 or GIF. The CLI is a budo dev server for hot
serving plus a browserify build for `--build` (a self-contained static page).

## Production Notes

- **It is beta and single-maintainer.** The API is stable in practice, but the
  repository has no unit tests, the promised API/CLI docs never fully
  materialized[^2], and commit velocity is low. Treat it as stable-but-quiet
  infrastructure, not an actively developed platform — do not expect fast issue
  turnaround (dozens remain open).
- **Node/npm version sensitivity.** The README pins expectations to `node@15`
  / `npm@7` or newer. The browserify-based CLI predates the ESM-only era, so
  modern packages that ship only ES modules or assume a webpack/Vite resolver
  can fail to bundle cleanly. Plain CommonJS dependencies are the safe path.
- **`ffmpeg` is an external dependency.** MP4 and GIF export require `ffmpeg`
  installed and on `PATH`. Frame-sequence export for long or high-resolution
  animations is slow — it renders and writes every frame to disk before muxing.
- **Global vs. `npx` drift.** The runtime library and the CLI version
  independently; a global CLI paired with a project-local `canvas-sketch` (or
  vice versa) can produce subtle mismatches. Keep both local per project.
- **Not a web-app framework.** `--build` produces a static page, but it renders
  a single sketch — no routing, hydration, or state model. Shipping interactive
  product UI with it is a category error.
- **Companion utilities are a separate install.** Randomness, math, geometry,
  and color helpers live in `canvas-sketch-util`, which most real sketches need.

## When to Use / When Not

**Use when:**
- You need exact print dimensions and DPI: gallery prints, risograph, posters.
- You make plotter/pen art and want SVG output aimed at an AxiDraw.
- You author seamless animation loops (`playhead`) for social or motion work.
- You want three.js/regl/p5 sketches with export ergonomics handled for you.
- You teach creative coding and want a low-ceremony authoring loop.

**Avoid when:**
- You are building an interactive web product or UI — this is an art tool.
- You require a modern ESM/Vite toolchain or tree-shaken production bundles.
- You need an actively maintained, versioned-past-1.0 dependency with SLAs.
- Your dependencies are ESM-only and refuse to bundle under browserify.

## Alternatives

- processing/p5.js — friendlier beginner API and a large learning ecosystem;
  use it when approachability matters more than print-DPI export and CLI
  scaffolding, which p5 does not provide out of the box.
- mrdoob/three.js — the 3D engine canvas-sketch wraps; use it directly when the
  deliverable is a real-time 3D scene rather than an exported artifact.
- openrndr/openrndr — Kotlin/JVM generative-art framework with strong plotter
  support; use it when you want compiled performance over a browser workflow.
- ojack/hydra — live-coding visual synth; use it for real-time/VJ performance
  rather than authoring fixed high-resolution output.
- abey79/vsketch — Python plotter-art framework; use it when your pipeline is
  Python and pen-plotting is the primary output.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-06 | First public release; framework + CLI split[^1]. |
| 0.x | 2018–present | Remained in `0.x` "beta"; export/units pipeline matured, roadmap items (docs, tests, GUI) largely unshipped[^2]. |
| — | 2026-06 | Repository still receiving occasional commits; no 1.0 tagged. |

## References

[^1]: mattdesl/canvas-sketch repository, created 2018-06-14; README describes it
  as "[beta] A framework for making generative artwork in JavaScript and the
  browser." https://github.com/mattdesl/canvas-sketch
[^2]: canvas-sketch README "Roadmap" section, listing still-outstanding items
  (API & CLI docs, unit tests, GUI/HUD controls, gallery mode).
  https://github.com/mattdesl/canvas-sketch/blob/master/README.md

## Tags

javascript, generative-art, creative-coding, canvas, webgl, print, plotter-art, browser, animation, art-tool
