# leeoniya/uPlot

> A small, fast Canvas 2D chart for time series, lines, areas, OHLC and bars — the library you reach for when render latency and bundle size matter more than features.

[GitHub repo](https://github.com/leeoniya/uPlot) ·
[License: MIT](https://github.com/leeoniya/uPlot/blob/master/LICENSE)

## Overview

uPlot is a single-author charting library (Leon Sorokin) first released in 2019[^1]. It is deliberately narrow: it plots time-series and numeric data on a Canvas 2D surface and does almost nothing else. There is no data parsing, no aggregation, no stacking, no animation, and no SVG fallback. That scope discipline is the entire point — the minified IIFE build is roughly 50 KB and the library renders on the order of 100,000 points per millisecond after a cold start[^2].

The defining tension is features versus footprint. Most charting libraries (Chart.js, ECharts, Highcharts, Plotly) grow toward being general visualization toolkits; uPlot refuses that path and stays a plotter. It has no WebGL or WASM path either, which the author frames as a feature: Canvas 2D has near-zero startup cost and no context limits, where WebGL charts pay a compile/upload tax on first paint[^2]. The practical consequence is that uPlot is one of the fastest options for the specific job of drawing many time-series points quickly, and a poor fit for anyone who wants a batteries-included charting UI.

The other cost is the API. uPlot is configuration-driven — you hand it a nested options object and a columnar data array — and the documentation is, in the author's own words, a "perpetual work in progress"[^3]. The real reference is the TypeScript definition file plus a large collection of runnable demos.

## Getting Started

```bash
npm install uplot
```

```js
import uPlot from "uplot";
import "uplot/dist/uPlot.min.css";

// data is COLUMNAR: [ xValues, series1, series2, ... ]
// x is unix seconds by default for a time scale
const data = [
  [1609459200, 1609462800, 1609466400],  // x (timestamps)
  [35, 71, 42],                            // series 1 y-values
  [90, 15, 60],                            // series 2 y-values
];

const opts = {
  width: 800,
  height: 400,
  series: [
    {},                                    // x-series (index 0) — required
    { label: "CPU", stroke: "red" },
    { label: "RAM", stroke: "blue" },
  ],
};

const plot = new uPlot(opts, data, document.body);

// live update: replace the whole data array
plot.setData(newData);
```

The columnar-not-row data layout is the most common first stumble: `data[0]` is always the x-axis, and each subsequent array is one series aligned by index.

## Architecture / How It Works

uPlot is a single Canvas 2D element with a thin DOM overlay for the legend, axis labels, cursor and selection box. It does not retain a scene graph — on each `setData` or resize it recomputes scale ranges and redraws the whole plot imperatively. There is no diffing and no virtual layer; speed comes from doing very little per point and batching path construction into as few `stroke()`/`fill()` calls as possible.

Data is columnar (structure-of-arrays) rather than an array of point objects. This is deliberate: typed, index-aligned arrays are cache-friendly and avoid per-point allocation, which is what lets uPlot iterate hundreds of thousands of points without GC pressure[^2].

The extension model is **hooks and plugins**. uPlot exposes lifecycle hooks (`init`, `setData`, `setScale`, `setCursor`, `drawSeries`, `draw`, and others); a plugin is just an object that registers into those hooks. Many capabilities that other libraries ship built-in are intentionally left to plugins or the host app — wheel zoom, touch zoom, panning, tooltips, and legend customization are all demonstrated as external code rather than core features[^3]. Path rendering itself is pluggable: `linear`, `spline`, `stepped`, and `bars` renderers are swappable, and you can supply your own.

Scales are the other core concept. A chart has named scales (default `x` and `y`); each series binds to a scale, and multiple y-scales let you overlay series with different units. Scales can be linear, uniform-log, or logarithmic, and time scales understand IANA time zones and DST when you supply a timezone-aware date formatter.

## Production Notes

- **Streaming has a ceiling.** uPlot can live-stream at 60fps and stays cheap doing it — the author measures ~10% CPU and ~12 MB updating 3,600 points at 60fps, versus 40%/77 MB for Chart.js and 70%/85 MB for ECharts on the same test[^2]. But it redraws the full in-view dataset each frame, so beyond ~100,000 in-view points it degrades. Past that, downsample before handing data to uPlot, drop the update rate, or move to a WebGL plotter (webgl-plot, TimeChart) as the README itself recommends[^2].

- **Canvas rasterization is hardware-dependent.** The same code can hit 57% CPU on one machine and 99% on another purely because of where the browser rasterizes Canvas2D. On Chromium, force-enabling `Canvas out-of-process rasterization` in `chrome://flags` produced a large framerate gain on integrated-GPU Linux for the author[^3]. This is a real deployment variable, not a micro-optimization — test on representative hardware.

- **No collision avoidance on axis labels.** uPlot does not reflow or hide overlapping tick labels. If you widen labels via custom formatters, you may have to hand-tune tick spacing metrics or labels will collide[^3].

- **The options object is the API, and it is under-documented.** New features and edge behavior are discovered through `/dist/uPlot.d.ts` and the demos rather than prose docs[^3]. Budget time to read the type definitions. Framework integration is community-maintained (React/Vue/Svelte wrappers by a third party, a separate Python binding)[^1] — none are first-party, so pin versions and expect to occasionally drop to the raw API.

- **Intentional non-features are permanent.** No stacked series, no animations, no built-in pan/drag, no data processing — these are stated design refusals, not roadmap gaps[^3]. If your requirements include them, you will be building on top of hooks indefinitely, and some (stacking) the author actively argues against.

## When to Use / When Not

**Use when:**
- You are plotting time series and initial render + cursor/zoom latency is the primary constraint.
- Bundle size matters (dashboards, embedded widgets, mobile web).
- You live-stream data and want low, steady CPU/RAM use.
- You are comfortable driving a configuration object and wiring interactions via hooks.

**Avoid when:**
- You want a high-level, batteries-included charting UI with tooltips, legends, and interactions out of the box — Chart.js or ECharts get you there faster.
- You need pie/radar/geo/sankey or other non-Cartesian chart types — uPlot is Cartesian time-series/line/bar only.
- You need to render millions of concurrently-visible points at high frame rates — use a WebGL/WebGPU plotter.
- Your team wants thorough prose documentation and first-party framework components.

## Alternatives

- chartjs/Chart.js — use when you want an easy, well-documented general chart library and can spend the extra bundle and render cost.
- apache/echarts — use when you need a broad chart-type catalog (maps, sankey, graph) and rich built-in interactions.
- plotly/plotly.js — use when you want scientific/statistical chart types and interactivity without hand-wiring.
- danchitnis/webgl-plot — use when you must render far more concurrent points than Canvas2D can handle in real time.
- huww98/TimeChart — use when you need WebGL-accelerated streaming time-series specifically.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial commit | 2019-09-27 | Repository created; Canvas2D time-series plotter[^1]. |
| 1.0.0 | 2020 | First stable major release. |
| 1.6.x | 2021–2024 | Long-lived 1.6 line; path renderers, log scales, timezone/DST, plugin API matured. |
| 1.6.24 | 2023 | Version used in the maintained benchmark suite[^2]. |

(uPlot has stayed on the 1.x line; the project favors incremental releases over major-version churn. Consult the repo's releases/CHANGELOG for exact per-version dates.)

## References

[^1]: leeoniya/uPlot — repository, README, and third-party integration list. https://github.com/leeoniya/uPlot
[^2]: uPlot README, "Introduction" and "Performance" sections — cold-start and streaming benchmarks (hardware-dated 2023-03-11). https://github.com/leeoniya/uPlot#performance
[^3]: uPlot README, "Non-Features", "Documentation", and "Unclog your rendering pipeline" sections. https://github.com/leeoniya/uPlot#non-features

## Tags

javascript, charting, data-visualization, time-series, canvas, streaming, performance, lightweight, plotting, line-chart
