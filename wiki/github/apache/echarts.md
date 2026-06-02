# apache/echarts

A feature-rich interactive charting library from Apache — Chinese-developed (originally by Baidu), donated to the Apache Foundation, used heavily in Chinese enterprise BI and globally for data dashboards.

## What it is

A JavaScript charting library with extensive chart types (line, bar, pie, scatter, heatmap, treemap, sunburst, geographic, 3D, networks, parallel coordinates, candlestick) and a declarative configuration API. Renders to Canvas or SVG. Apache 2.0 licensed under Apache Software Foundation governance.

## Key features

- 20+ chart types including specialized ones (sankey, gauge, themeRiver, candlestick, parallel).
- Declarative `option` config — describe what you want, ECharts handles drawing.
- Canvas + SVG renderers selectable per chart.
- WebGL extension (`echarts-gl`) for 3D charts and large-scale rendering.
- i18n built in; locale files for many languages.
- Responsive + dynamic data — `setOption()` patches the chart in place.
- Apache 2.0 licensed under the ASF.

## Tech stack

- TypeScript primary.
- Canvas + SVG render backends.
- npm package: `echarts`.

## When to reach for it

- You're building data dashboards with a rich chart type palette.
- You need specialized chart types (sankey, candlestick, geographic) not in lighter libraries.
- You want ASF-governed library stability.

## When *not* to reach for it

- Bundle size matters — ECharts is heavy; use Chart.js or lighter alternatives for simple cases.
- You're React-native and want components — `apache-superset/superset` UIs are React but ECharts itself is framework-agnostic; wrappers exist (echarts-for-react) but they're community-maintained.

## Maturity signal

66k stars, 20k forks, Apache 2.0, actively maintained. 12+ years; Apache project since 2018. Open-issues count of 1,624 tracks per-chart-type feature requests.

## Alternatives

- Chart.js — simpler, lighter, fewer chart types.
- D3 — lower-level primitives.
- Plotly.js — broader scientific-charting focus.

## Tags

typescript, data-visualization, chart, library, apache-license, frontend, svg, canvas, echarts
