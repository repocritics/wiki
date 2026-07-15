# apexcharts/apexcharts.js

> Interactive SVG charts for the browser — 16+ chart types, framework-agnostic, and, since v5, no longer MIT-licensed.

[GitHub repo](https://github.com/apexcharts/apexcharts.js) ·
[Official website](https://apexcharts.com) ·
License: revenue-based source-available (not OSI-approved)

## Overview

ApexCharts is a client-side charting library that renders interactive charts as SVG. It ships a single `ApexCharts` class configured by one large options object, has zero runtime dependencies, and works in vanilla JS or through official wrappers for React, Vue, Angular, Blazor, and Stencil[^1]. First published in 2018, it occupies the middle of the charting spectrum: more declarative and batteries-included than D3, lighter and more opinionated than a full BI toolkit, and aimed squarely at dashboards and data-heavy product UIs.

The defining fact about ApexCharts in 2026 is not technical but legal. Through v4 the library was MIT-licensed. With **v5.0.0 (mid-2025) the project relicensed to a revenue-based, source-available license**: free for individuals and organizations under $2M USD annual gross revenue, and requiring a paid commercial license at or above that threshold[^2]. GitHub reports the license as `NOASSERTION` because it is a custom, non-OSI license. Any evaluation of ApexCharts today must start here — the code is still public and installable via npm, but it is no longer open source in the OSI sense, and pinning to a `4.x` release is the only way to stay on MIT terms.

Setting the license aside, ApexCharts is a mature, well-documented library whose main technical draw is a broad chart-type catalog behind one consistent configuration API, with SSR and tree-shaking added in the v5 line.

## Getting Started

```bash
npm install apexcharts
```

```js
import ApexCharts from 'apexcharts'

const chart = new ApexCharts(document.querySelector('#chart'), {
  chart: { type: 'bar' },
  series: [{ name: 'Sales', data: [30, 40, 35, 50, 49, 60, 70, 91, 125] }],
  xaxis: { categories: [1991, 1992, 1993, 1994, 1995, 1996, 1997, 1998, 1999] },
})

chart.render()
```

Everything is driven by the options object passed to the constructor. Updating a live chart is done imperatively via `chart.updateOptions(...)` and `chart.updateSeries(...)` rather than by re-rendering, which matters for how the framework wrappers are built (see below).

## Architecture / How It Works

ApexCharts is a monolithic runtime around a central `ApexCharts` class. Rendering targets SVG (via the SVG.js lineage internally), not Canvas or WebGL — every data point becomes a DOM node. That choice is what makes charts crisp, CSS-styleable, and trivially exportable to SVG/PNG, and it is also the primary scaling ceiling: DOM node count grows with data volume.

The public surface is deliberately narrow: construct with an options object, then mutate through a handful of methods (`render`, `updateSeries`, `updateOptions`, `destroy`, plus toolbar/zoom helpers). The options object is deep and stringly-typed — `chart.type`, `plotOptions`, `dataLabels`, `xaxis`, `yaxis`, `annotations`, and so on — which makes discoverability dependent on the docs rather than the type system, even though TypeScript definitions ship with the package.

The v5 line reworked packaging around two additions:

- **SSR entry points.** `apexcharts/ssr` exposes `renderToHTML()` to produce hydration-ready SVG on the server, paired with `apexcharts/client`'s `hydrate()` / `hydrateAll()`. This replaces the older meta-framework workaround of importing the chart dynamically with `{ ssr: false }`[^1].
- **Tree-shakable entry points.** `apexcharts/core` provides the bare class with no chart types or features; individual types (`apexcharts/bar`, `apexcharts/line`, …) and features (`apexcharts/features/toolbar`, `.../annotations`, `.../exports`) are opted in by import[^3]. The default `import ApexCharts from 'apexcharts'` still pulls in everything.

The official framework wrappers (`react-apexcharts`, `vue3-apexcharts`, `ng-apexcharts`, etc.) are thin adapters that translate declarative props into the imperative `update*` calls. They lag the core library on releases and are the usual source of framework-specific bugs; the core is where the actual work happens.

## Production Notes

**The license is the biggest production risk, not a footnote.** If your organization is at or above $2M annual revenue, `npm install apexcharts` at v5+ pulls in code that requires a commercial license for use[^2]. Teams that adopted ApexCharts under MIT should audit their pinned version: `4.x` remains MIT, but any `^` range or a fresh install lands on v5+ under the new terms. There is no automated enforcement, but the obligation is real and legal, not advisory.

**SVG DOM cost scales with data.** Because each point is a DOM element, dense series (thousands of points, real-time streaming, many synchronized charts) can produce sluggish interaction and high memory use. ApexCharts is a good fit for dashboard-scale data (tens to low thousands of points); for large scatter/time-series or high-frequency updates, a Canvas/WebGL library is a better structural match.

**Imperative updates and framework churn.** In React/Vue, feeding new data means the wrapper diffs props and calls `updateSeries`/`updateOptions`. Passing a fresh options object identity on every render, or mutating series in place, are common causes of flicker, lost zoom state, or full re-renders. Memoize options and pass stable references.

**Wrapper release lag.** Core, React, Vue, and Angular packages version independently. A core feature or fix is frequently unavailable through a wrapper for some time, and peer-dependency ranges can block upgrades. Check the specific wrapper's changelog, not just the core changelog.

**Styling and theming** are done through the options object and CSS, not a design-token system; consistent theming across many charts usually means factoring out a shared base-options object in your own code.

## When to Use / When Not

**Use when:**
- You need a broad catalog of standard chart types (line/area/bar/pie/heatmap/candlestick/radar/treemap) behind one consistent config API.
- You're building product dashboards or SaaS UIs with dashboard-scale data volumes.
- You want SSR/hydration and tree-shaking in a framework-agnostic library with first-party wrappers.
- Your organization is under the $2M revenue threshold, or you're willing to buy a commercial license.

**Avoid when:**
- You require a permissive OSI license — v5+ is not open source; you'd be pinned to unmaintained `4.x` MIT releases.
- You render very large datasets or high-frequency real-time streams — SVG DOM cost will bite.
- You need bespoke, non-standard visualizations — a low-level toolkit (D3, visx) gives control ApexCharts' options object cannot express.
- You want the smallest possible footprint for a single simple chart — lighter libraries exist.

## Alternatives

- chartjs/Chart.js — MIT, Canvas-based; better for large/streaming data and a smaller API surface, at the cost of SVG crispness and export ease.
- d3/d3 — MIT, low-level primitives; use when you need full control over bespoke visualizations rather than preset chart types.
- apache/echarts — Apache-2.0, Canvas/SVG, very broad feature set; use for large datasets and complex interactive analytics dashboards.
- plotly/plotly.js — MIT, scientific/statistical charting; use when you need 3D, statistical chart types, or Python/R parity.
- observablehq/plot — ISC, concise grammar-of-graphics on top of D3; use for exploratory/analytical charts with terse declarative code.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018 | Initial public release; SVG charts, MIT-licensed[^1]. |
| 3.0 | 2019 (approx.) | Major API revision; long-lived 3.x line. |
| 4.0 | 2024-10-29 | Major release; still MIT-licensed[^4]. |
| 5.0 | 2025-07 | SSR entry points, tree-shaking, and relicense to revenue-based source-available[^2]. |
| 5.16.0 | 2026-07-03 | Latest release in the 5.x line[^4]. |

## References

[^1]: ApexCharts README and documentation. https://apexcharts.com/docs/
[^2]: ApexCharts license terms — revenue-based, free under $2M USD annual gross revenue, commercial license required at or above. https://apexcharts.com/license
[^3]: ApexCharts tree-shaking guide — `apexcharts/core` plus per-type and per-feature entry points. https://apexcharts.com/docs/tree-shaking/
[^4]: GitHub Releases, apexcharts/apexcharts.js — v4.0.0 (2024-10-29), v5.16.0 (2026-07-03). https://github.com/apexcharts/apexcharts.js/releases

## Tags

javascript, typescript, charting, data-visualization, svg, dashboard, frontend, source-available, ssr, framework-agnostic
