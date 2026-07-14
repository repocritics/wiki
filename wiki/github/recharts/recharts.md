# recharts/recharts

> Composable, SVG-rendered React charts that use D3 for the math and React components for everything you see.

[GitHub repo](https://github.com/recharts/recharts) ·
[Official website](https://recharts.github.io) ·
[License: MIT](https://github.com/recharts/recharts/blob/main/LICENSE)

## Overview

Recharts is a charting library for React, first published in 2015[^1]. Its
defining idea is that a chart is assembled from ordinary React components:
`<LineChart>` holds `<XAxis>`, `<CartesianGrid>`, `<Tooltip>`, and one or more
`<Line>` elements, and each is a real component with its own props. This reads
cleanly, composes with JSX, and is why Recharts remains one of the most widely
installed React chart libraries — roughly 27k GitHub stars and among the
highest weekly npm download counts in its category as of 2026.

The tradeoff behind that ergonomics is subtle and worth understanding before
adopting it. Recharts does not simply render the children you pass; it
*introspects* them. The parent chart walks its `children`, reads each child's
component type and props as configuration, and then draws the actual SVG
itself. D3 modules (`d3-scale`, `d3-shape`, `d3-interpolate`) compute scales and
path geometry; Recharts owns the DOM. This is the source of both its
composability and its most common frustrations: children behave as declarative
config, not as free-form render slots.

Recharts is a good fit for dashboards, admin panels, and product analytics
where charts are relatively standard (lines, bars, areas, pies, scatter,
radial) and data volumes are modest. It is a poor fit when you need tens of
thousands of points, non-standard chart types, or pixel-level custom rendering.

## Getting Started

```sh
npm install recharts react-is
```

`react-is` is a peer dependency and its major version must match your installed
`react` version[^2] — a version mismatch is a frequent first-run failure.

```jsx
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip,
         ResponsiveContainer } from "recharts";

const data = [
  { name: "Jan", uv: 400, pv: 240 },
  { name: "Feb", uv: 300, pv: 456 },
  { name: "Mar", uv: 200, pv: 139 },
];

export default function Chart() {
  return (
    <ResponsiveContainer width="100%" height={320}>
      <LineChart data={data}>
        <CartesianGrid stroke="#f5f5f5" />
        <XAxis dataKey="name" />
        <YAxis />
        <Tooltip />
        <Line type="monotone" dataKey="uv" stroke="#ff7300" />
        <Line type="monotone" dataKey="pv" stroke="#387908" />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

## Architecture / How It Works

The core pattern is **component-as-configuration**. A chart container
(`LineChart`, `BarChart`, `ComposedChart`, etc.) iterates over
`React.Children`, matches each child against known display names (`Line`,
`XAxis`, `Bar`, `Tooltip`…), and extracts its props into an internal model.
From that model it computes scales via D3 and emits the `<svg>` tree. The child
elements you write are largely markers describing *what* to draw, not literal
render output.

This introspection has direct consequences:

- **Wrapping breaks recognition.** If you wrap `<Line>` in your own component,
  the parent no longer sees a `Line` in its children and silently drops it.
  Custom wrappers must forward the right type or the chart ignores them. This is
  the single most-reported point of confusion for new users.
- **Order and nesting matter** in ways plain React does not usually impose.
- **Sizing is measured, not flowed.** `ResponsiveContainer` uses a
  `ResizeObserver` on its parent; if the parent has no resolved height, the
  chart renders at zero height and appears blank.

Animation is handled by the maintainers' own `react-smooth` library rather than
CSS transitions, which is why animation glitches on rapid re-render are a
recurring theme. Rendering is SVG-only — there is no canvas or WebGL backend —
so the number of DOM nodes scales with the number of data points and series.

Recharts 3 (2025) reworked internals substantially, moving shared chart state
into an internal store (Redux-based) and adding a more explicit accessibility
layer[^3]. The public composition API stayed broadly similar, but some legacy
props and behaviors changed; treat a 2→3 upgrade as a real migration.

## Production Notes

- **Container height is the #1 footgun.** A chart inside a flex/grid cell with
  no explicit or resolved height renders blank. Give `ResponsiveContainer` a
  fixed pixel `height`, or ensure the parent has a real height.
- **SVG DOM cost.** Every data point becomes DOM. A few hundred points per
  series is comfortable; a few thousand degrades interaction and layout. For
  large series, downsample/aggregate before rendering, or move to a
  canvas-based library. There is no built-in virtualization.
- **Disable animation for live/updating data.** `isAnimationActive={false}` on
  series avoids flicker and re-animation on every data change, and measurably
  reduces re-render cost.
- **`react-is` mismatch** produces cryptic element-type errors; pin it to your
  React major.
- **Tooltip/legend re-renders.** Custom `content` render functions run
  frequently; keep them cheap and memoized or they dominate hover latency.
- **Bundle size.** Recharts pulls in several D3 submodules. Tree-shaking has
  improved across versions but the baseline is heavier than minimal
  alternatives; check your bundle if size is a constraint.
- **Upgrades.** The 2→3 transition changed internals and some APIs; test
  interactivity (tooltip, legend toggling, custom shapes) after upgrading
  rather than assuming a drop-in.

## When to Use / When Not

**Use when:**
- You want declarative, JSX-composed charts inside a React app.
- Your charts are standard types with modest data volumes (dashboards, KPIs).
- You value readable markup and don't need a canvas renderer.
- You want SVG output (crisp at any zoom, stylable, inspectable).

**Avoid when:**
- You need to render tens of thousands of points or real-time high-frequency
  streams — reach for a canvas/WebGL library.
- You need chart types Recharts doesn't ship, or fully custom rendering — a
  lower-level primitive library gives more control.
- You're not on React, or want a framework-agnostic charting core.
- Bundle size is tightly budgeted and you only need one simple chart.

## Alternatives

- nivo (plouc/nivo) — use instead when you want richer built-in chart types,
  strong theming, and optional canvas/server rendering out of the box.
- visx (airbnb/visx) — use instead when you want low-level D3 primitives as
  React components and are willing to assemble charts yourself for full control.
- apache/echarts — use instead when you need canvas performance on very large
  datasets and don't require a React-native component API.
- chartjs/Chart.js — use instead when you want a lightweight, canvas-based,
  framework-agnostic charting library with an imperative config.
- d3/d3 — use instead when you need complete rendering control and are prepared
  to build the chart layer from scratch.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-08 | Repository created; SVG + React + D3 composition model[^1]. |
| 1.x | ~2018 | Stabilized the composable component API. |
| 2.x | ~2021 | Long-lived major line; broad ecosystem adoption[^4]. |
| 3.x | 2025 | Internal state store (Redux-based), accessibility layer, API cleanup[^3]. |

## References

[^1]: recharts/recharts repository — created 2015-08-07 (GitHub API metadata).
https://github.com/recharts/recharts
[^2]: Recharts README, Installation — `react-is` must match the installed
`react` version. https://github.com/recharts/recharts#installation
[^3]: Recharts releases — 3.0 internal rewrite and accessibility changes.
https://github.com/recharts/recharts/releases
[^4]: Recharts documentation and storybook. https://recharts.github.io

## Tags

react, charts, data-visualization, svg, d3, typescript, dashboard, frontend, ui-components, charting-library
