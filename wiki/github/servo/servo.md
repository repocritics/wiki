# servo/servo

> An independent, parallelism-oriented web engine written in Rust — the research
> codebase that seeded Firefox's Stylo and WebRender, now embedding-focused.

[GitHub repo](https://github.com/servo/servo) ·
[Official website](https://servo.org) ·
[License: MPL-2.0](https://github.com/servo/servo/blob/main/LICENSE)

## Overview

Servo is a web rendering engine — the layer that parses HTML/CSS, runs layout,
and paints pixels — not a full browser product. It began at Mozilla Research in
2012 as the flagship application driving Rust's own design[^1]: the two projects
co-evolved, and much of Rust's early ownership and concurrency story was
pressure-tested against the demands of a parallel browser engine. Servo's stated
thesis was that memory-safe systems code plus task-parallel layout and styling
could exploit multi-core hardware in ways Gecko and WebKit's largely
single-threaded engines could not.

Servo's most consequential output is arguably not Servo itself but the
components it fed upstream. **Stylo** (Servo's parallel CSS style system) and
**WebRender** (its GPU-based display-list renderer) were both integrated into
Firefox as part of Project Quantum, shipping in Firefox 57 in late 2017[^2].
Servo also produced a family of widely reused Rust crates — `html5ever`,
`cssparser`, `selectors`, `smallvec`, `string_cache` — that now underpin
projects far outside the browser world.

The defining tension today is scope versus staffing. Mozilla laid off the Servo
team in August 2020 and the project was transferred to the Linux Foundation[^3].
It sat largely dormant until 2023, when Igalia took over stewardship and
restarted active development under a new governance model[^4]. The engine is
genuinely alive again and lands features weekly, but it remains a
web-platform-incomplete engine chasing a two-decade spec surface with a fraction
of the resources of Blink or WebKit. Treat it as an embeddable engine and a
research/alternative-engine effort, not a drop-in replacement for a production
browser.

## Getting Started

Servo builds from source via `mach`, a Python build orchestrator inherited from
Mozilla. There is no `cargo install`; you clone and bootstrap. Nightly binaries
of the `servoshell` demo browser are published at servo.org.

```bash
git clone https://github.com/servo/servo
cd servo

# uv (Python) and rustup must be installed first
./mach bootstrap        # installs system + toolchain dependencies
./mach build --release  # cold builds take 10+ minutes
./mach run https://servo.org
```

Embedding Servo as a library uses the `WebView` API surface exposed by the
`libservo` crate; the `servoshell` application in-tree is the reference consumer
and the most reliable embedding example, since the embedding API is still
evolving and not yet stability-guaranteed.

## Architecture / How It Works

Servo is a multi-crate workspace. The engine is split into cooperating
components that communicate largely by message passing rather than shared state:

- **Parsing** — `html5ever` (HTML) and the `cssparser`/`selectors` stack (CSS)
  turn bytes into a DOM and stylesheet rules.
- **Style** — the parallel style system (`style`, the crate upstreamed to
  Firefox as Stylo) computes cascaded/computed values across DOM nodes in
  parallel using the `rayon` work-stealing thread pool.
- **Layout** — Servo has carried more than one layout engine. The modern layout
  engine (historically "layout 2020") implements the current CSS box model,
  flexbox, and increasingly grid; it replaced the older experimental layout code
  as the default.
- **Scripting** — JavaScript is not Servo's own; it runs **SpiderMonkey** via
  the `mozjs` bindings. The DOM is implemented in Rust and bound to SpiderMonkey
  through generated bindings, which is a substantial share of the `script` crate.
- **Rendering** — **WebRender** turns a display list into GPU draw calls
  (OpenGL/OpenGL ES/ANGLE), treating the page more like a scene graph submitted
  to the GPU than a sequence of CPU raster operations.
- **Embedding/shell** — `libservo` exposes the `WebView` API; `servoshell` is
  the demo application wrapping it with windowing (winit) and UI.

The recurring architectural bet is parallelism and process/thread isolation
between pipelines (one per browsing context). This is Servo's differentiator and
also its cost: message-passing boundaries and multi-crate separation make the
codebase large and the build slow, and mean that features touching several
stages (e.g. a new CSS property that affects style, layout, and paint) require
coordinated changes across crate boundaries.

## Production Notes

**This is not a browser you ship to end users.** Web-platform coverage is
partial: expect missing or incomplete support across large CSS/DOM/JS surface
areas, and sites that assume Blink/WebKit quirks will often break. Servo is
appropriate as an embeddable engine for controlled content, as a rendering
component in Rust applications, or as a research/reference engine — not as a
general-purpose user agent.

**Build cost is real.** As a large Rust workspace with C/C++ dependencies
(SpiderMonkey, ANGLE, and platform graphics stacks), cold builds run 10+ minutes
and pull significant system dependencies through `./mach bootstrap`. `mach`
wraps `cargo` but adds Servo-specific build steps; invoking `cargo build`
directly is not the supported path.

**No versioned releases.** Servo ships nightlies, not semver releases. There is
no stable API contract for embedders yet — the `WebView` embedding surface
changes without deprecation cycles, so pin to a specific commit and expect to
track breaking changes when you upgrade.

**GPU and platform variance.** WebRender's reliance on the GPU means rendering
correctness and performance depend on the graphics driver and backend
(native GL vs ANGLE). Headless and CI rendering setups need explicit software or
ANGLE configuration; behavior on exotic drivers is a known source of bugs.

**Platform breadth is recent and uneven.** Android and OpenHarmony targets have
been added and are actively worked on, but they are newer than the
macOS/Linux/Windows desktop targets and correspondingly less mature.

**Governance risk is a legitimate input.** The 2020–2023 dormancy is a cautionary
data point. Current momentum under Igalia and Linux Foundation stewardship is
real and visible in commit activity, but funding depends on sponsorship and
grants rather than a platform vendor's product roadmap.

## When to Use / When Not

**Use when:**
- You want a memory-safe, embeddable web engine written in Rust for rendering
  controlled or semi-trusted content inside a native app.
- You're building or researching an alternative to the Blink/WebKit/Gecko
  monoculture and want a genuinely independent engine.
- You need the Servo-derived crates (`html5ever`, `cssparser`, `selectors`) —
  these are production-grade even where the full engine is not.

**Avoid when:**
- You need a spec-complete browser that renders the arbitrary web reliably today.
- You need a stable, guaranteed embedding API with deprecation cycles.
- You can't absorb from-source builds, nightly-only distribution, and
  upgrade-time breakage.
- You're targeting a platform outside the tested desktop tier and need maturity.

## Alternatives

- chromium/chromium — the Blink engine; spec-complete and dominant. Use when you
  need to render the real web reliably and can accept the footprint.
- WebKit/WebKit — Apple's engine (Safari, WKWebView). Use when you're on
  Apple platforms or want the second most-complete engine.
- LadybirdBrowser/ladybird — independent browser + engine in C++ with no
  third-party engine dependency. Use when you want another from-scratch engine
  effort with a browser-product focus.
- servo/servo consumers like versotile-org/verso — a windowing/browser layer
  built on `libservo`. Use when you want Servo's engine with a higher-level
  embedding/browser shell already assembled.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Project start | 2012 | Mozilla Research; Rust's flagship driving application[^1]. |
| Stylo + WebRender in Firefox | 2017-11 | Upstreamed via Project Quantum, Firefox 57[^2]. |
| Mozilla layoffs | 2020-08 | Servo team cut; project moved to the Linux Foundation[^3]. |
| Dormancy | 2020–2022 | Minimal activity after transfer. |
| Igalia revival | 2023 | New stewardship and governance; active development resumes[^4]. |
| Layout + platform push | 2024–2026 | Modern layout engine default; Android/OpenHarmony targets; weekly nightlies. |

## References

[^1]: Servo README and project history; Servo as Rust's early flagship application. https://github.com/servo/servo
[^2]: Mozilla, "Firefox Quantum" (Firefox 57) — Stylo and WebRender integration, 2017-11. https://blog.mozilla.org/en/products/firefox/introducing-firefox-quantum/
[^3]: Linux Foundation, "Servo Joins the Linux Foundation", 2020. https://www.linuxfoundation.org/press/press-release/servo-to-advance-under-linux-foundation
[^4]: The Servo Project / Igalia — Servo revival and governance, 2023. https://servo.org/blog/2023/01/16/servo-2023/

## Tags

rust, web-engine, browser-engine, rendering-engine, layout, webrender, stylo, embeddable, parallelism, servo, cross-platform
