# MakieOrg/Makie.jl

> Julia's GPU-accelerated plotting ecosystem — one plotting API, four swappable backends, from raytraced 3D to publication vector graphics.

[GitHub repo](https://github.com/MakieOrg/Makie.jl) ·
[Official website](https://docs.makie.org/stable) ·
[License: MIT](https://github.com/MakieOrg/Makie.jl/blob/master/LICENSE)

## Overview

Makie is the plotting and interactive visualization ecosystem for Julia, started by Simon Danisch in 2017 as an outgrowth of his GLVisualize work[^1]. The core package defines the plotting API, a reactive attribute system, and a constraint-based layout engine; rendering is delegated to backend packages living in the same monorepo: **GLMakie** (OpenGL, native windows, interactive 3D), **CairoMakie** (Cairo, static 2D vector output for publications), **WGLMakie** (WebGL in browsers/notebooks via Bonito), and **RPRMakie** (RadeonProRender raytracing). The same script renders on any of them by swapping one `using` line.

At 2.8k stars it is mid-sized by GitHub standards but a flagship package of the Julia ecosystem, effectively co-default (with Plots.jl) for scientific plotting there. Development is very active — pushes land daily, and the 1,000+ open issues reflect a large surface area (four renderers, layout, text, interaction) more than neglect. It has a peer-reviewed JOSS paper (2021)[^2] and BMBF (German federal) funding history.

The defining tension: Makie built a full retained-mode graphics stack — its own text rendering, layouting, event handling, and GPU pipelines — rather than binding an existing library like matplotlib. That buys GPU-speed interactivity and backend uniformity; it costs compile latency, 0.x-forever API churn, and a smaller recipe ecosystem than matplotlib's.

## Getting Started

Install a backend; each one re-exports the whole Makie API:

```julia
using Pkg
Pkg.add("CairoMakie")   # or GLMakie / WGLMakie / RPRMakie
```

```julia
using CairoMakie

x = range(0, 10; length = 200)
fig = Figure(size = (700, 400))
ax = Axis(fig[1, 1]; xlabel = "x", ylabel = "y", title = "Waves")
lines!(ax, x, sin.(x); label = "sin")
scatter!(ax, x[1:10:end], cos.(x[1:10:end]); label = "cos")
axislegend(ax; position = :rb)
save("waves.png", fig)   # or .svg / .pdf with CairoMakie
fig
```

## Architecture / How It Works

**Scene graph + Blocks.** A `Figure` owns a `GridLayout` of *Blocks* — `Axis`, `Axis3`, `Colorbar`, `Legend`, `Slider`, `Menu`, etc. Blocks size themselves via a constraint solver (the former MakieLayout package, merged into core in 2021 alongside the rename from AbstractPlotting to Makie v0.13)[^3]. Under the Blocks sits a `Scene` tree holding primitive plot objects (`Scatter`, `Lines`, `Mesh`, `Heatmap`, …) with camera and projection state.

**Reactivity via Observables.** Every plot attribute is (historically) an `Observable`; `lift`/`map` chains propagate changes, so updating one Observable re-renders only what depends on it. This powers animations (`record`) and interactive GUIs without a separate framework. Since v0.24 (2025) the per-attribute Observable network inside plot objects is being replaced by a ComputePipeline compute graph, which deduplicates updates and cuts memory — mostly invisible to users, breaking for code that reached into plot internals[^4].

**Recipes.** `@recipe` lets any package define new plot types (`myplot(data)`) that decompose into Makie primitives, without depending on a backend. This is how AlgebraOfGraphics.jl (grammar-of-graphics), GraphMakie, GeoMakie, and dozens of domain packages plug in.

**Backends** implement rendering for the primitive set only. GLMakie keeps data on the GPU via GLSL shaders (millions of scatter points at interactive rates); CairoMakie draws through Cairo with a painter's-order sort (no depth buffer — its 3D is best-effort); WGLMakie serializes scenes to the browser over Bonito.jl; RPRMakie hands geometry to AMD's raytracer for offline-quality stills. Text is rendered in-house (FreeType glyph atlases); LaTeX strings go through MathTeXEngine — a pure-Julia subset of TeX math, not a full LaTeX install, so some macros silently do not work.

## Production Notes

**Latency ("time to first plot").** Historically the #1 complaint: a minute+ of compile time before the first figure. Julia 1.9's native code caching plus precompilation work in Makie cut this to a few seconds on current versions[^5]. Still, every Makie version bump triggers a long precompile of the whole stack; caching `~/.julia/compiled` in CI is effectively mandatory.

**Headless environments.** GLMakie requires OpenGL 3.3+ and a display; on servers/CI it fails unless run under `xvfb-run`. The pragmatic rule: CairoMakie for CI and papers, GLMakie for local exploration, WGLMakie for remote/notebook use.

**CairoMakie and large data.** Vector output embeds every primitive: a 10^6-point scatter produces viewer-choking PDFs/SVGs. Rasterize heavy plots (`rasterize = true` per-plot) or render those panels in GLMakie.

**WGLMakie state.** Browser-side plots depend on a live Bonito server; reconnects and notebook re-runs can orphan widget state — fine for dashboards you control, rough for long-lived deployments.

**0.x churn.** Makie has never shipped 1.0. Breaking releases arrive roughly twice a year; v0.20 (Nov 2023) renamed `resolution` to `size` and touched many defaults[^6]. The recipe ecosystem lags: after each breaking release expect weeks where GeoMakie/GraphMakie/AlgebraOfGraphics compat bounds hold your environment back. Pin Makie in `[compat]` and upgrade deliberately.

**Animations** (`record(fig, "out.mp4", frames)`) ship ffmpeg via FFMPEG.jl — no system install needed — but encoding options beyond the basics mean dropping to raw ffmpeg flags.

## When to Use / When Not

**Use when:**
- You work in Julia and need interactive 3D or GPU-scale scatter/mesh/volume rendering next to your simulation code.
- You want one API that produces both an interactive exploration window and a publication-quality vector figure.
- You are building small scientific GUIs (sliders, menus, live-updating plots) without leaving Julia.

**Avoid when:**
- You are not otherwise using Julia — the runtime and precompile cost is not worth importing for plotting alone.
- You need quick throwaway plots with minimal latency budget — Plots.jl with a lightweight backend, or gnuplot, is lighter.
- You need strict long-term API stability (archived reproducible scripts): 0.x breaking cadence works against you unless you pin manifests.
- You need full LaTeX typesetting in figures — MathTeXEngine covers a subset; PGFPlots/pgf-based tooling is stronger there.

## Alternatives

- JuliaPlots/Plots.jl — use instead when you want a stable, minimal-latency meta-API over multiple backends (GR, PythonPlot, …) and don't need interactivity or GPU scale.
- MakieOrg/AlgebraOfGraphics.jl — not a competitor but the grammar-of-graphics layer on top of Makie; use for tabular/statistical plotting in ggplot2 style.
- GiovineItalia/Gadfly.jl — pure-Julia grammar-of-graphics; use for classic static statistical graphics, accepting slower maintenance.
- queryverse/VegaLite.jl — use when you want declarative JSON-spec charts that render identically in browsers and notebooks.
- matplotlib/matplotlib (via PythonPlot.jl) — use when you need matplotlib's ecosystem depth and documented recipes more than Julia-native speed.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.x | 2017-09 | Repo created; grew out of GLVisualize[^1]. |
| 0.13 | 2021-04 | AbstractPlotting renamed/merged into Makie; MakieLayout (Figure/Axis/Blocks) absorbed into core[^3]. |
| — | 2021-09 | JOSS paper published[^2]. |
| — | 2022 | Org moved from JuliaPlots to MakieOrg; monorepo consolidation of backends. |
| 0.19 | 2023-01 | Precompilation overhaul riding Julia 1.9 pkgimages; large latency drop[^5]. |
| 0.20 | 2023-11 | Breaking: `resolution` → `size`, theme/default changes[^6]. |
| 0.21 | 2024-05 | WGLMakie on Bonito.jl; further latency and rendering work. |
| 0.24 | 2025 | ComputePipeline begins replacing Observable-based plot internals[^4]. |

## References

[^1]: Simon Danisch, GLVisualize → Makie lineage; JuliaCon talks 2017–2018. https://github.com/JuliaGL/GLVisualize.jl
[^2]: Danisch & Krumbiegel, "Makie.jl: Flexible high-performance data visualization for Julia", JOSS 6(65), 3349, 2021. https://doi.org/10.21105/joss.03349
[^3]: Makie.jl v0.13 release (AbstractPlotting merge). https://github.com/MakieOrg/Makie.jl/releases
[^4]: Makie.jl CHANGELOG — v0.24, ComputePipeline. https://github.com/MakieOrg/Makie.jl/blob/master/CHANGELOG.md
[^5]: Julia 1.9 highlights — native code caching (pkgimages). https://julialang.org/blog/2023/04/julia-1.9-highlights/
[^6]: Makie.jl v0.20.0 release notes. https://github.com/MakieOrg/Makie.jl/releases/tag/v0.20.0

## Tags

julia, data-visualization, plotting, gpu, opengl, webgl, scientific-computing, graphics, interactive-visualization, vector-graphics
