# FormidableLabs/victory

> Composable React components that wrap D3's math to render interactive SVG charts.

[GitHub repo](https://github.com/FormidableLabs/victory) ·
[Official website](https://commerce.nearform.com/open-source/victory/) ·
[License: MIT](https://github.com/FormidableLabs/victory/blob/main/LICENSE.txt)

## Overview

Victory is a charting library for React, first published in 2015 by Formidable
Labs[^1]. Its organizing idea is composition: instead of a single `<Chart>` with
a large configuration object, you assemble a chart from small components —
`VictoryChart`, `VictoryBar`, `VictoryLine`, `VictoryAxis`, `VictoryScatter`,
`VictoryPie`, and so on — that share a coordinate system. Each renders SVG, and
React owns the DOM. Victory uses D3 (the `d3-scale`, `d3-shape`, `d3-interpolate`
family of modules) for the mathematics of scales, paths, and interpolation, but
does none of D3's imperative DOM manipulation[^2].

The audience is React and React Native developers who want charts that look and
behave like ordinary components — styled with props, wired with the same event
model, animated declaratively — rather than an embedded imperative visualization
engine. The defining tradeoff follows directly from that choice: because every
data point becomes real SVG DOM nodes, Victory is pleasant and flexible at small
to medium data volumes and degrades sharply at large ones. It is a component
library, not a rendering engine.

Formidable, Victory's original maintainer, was absorbed into NearForm; the
project's documentation and homepage now live under `commerce.nearform.com`[^3].
The repository remains actively maintained — releases ship regularly and the
current line is v37[^4] — though it is a mature library whose API has been stable
for years rather than one under rapid expansion.

## Getting Started

```sh
npm install victory
# React Native: npm install victory-native
```

```jsx
import { VictoryChart, VictoryBar, VictoryTheme } from "victory";

const data = [
  { quarter: "Q1", earnings: 13000 },
  { quarter: "Q2", earnings: 16500 },
  { quarter: "Q3", earnings: 14250 },
  { quarter: "Q4", earnings: 19000 },
];

export default function Earnings() {
  return (
    <VictoryChart theme={VictoryTheme.material} domainPadding={20}>
      <VictoryBar data={data} x="quarter" y="earnings" />
    </VictoryChart>
  );
}
```

`VictoryChart` is the coordinating parent: it computes a shared domain and scale
from its children and passes them down, so the axis and the bars agree on layout
without you wiring the numbers by hand. Any single component (`<VictoryPie />`)
also renders standalone.

## Architecture / How It Works

The core mechanism is **prop injection through a container**. `VictoryChart`
(and `VictoryGroup`, `VictoryStack`) inspect their children, collect each
child's data domain, reconcile a shared `domain` and `scale`, and clone the
children with those resolved props. This is why children need no explicit
coordinate configuration — the parent injects it. It also means components used
this way must be Victory-aware; arbitrary children do not participate.

**D3 as a library, not a framework.** Victory imports discrete D3 modules —
`d3-scale` for mapping data to pixels, `d3-shape` for arc/line/area path
generators, `d3-interpolate` and `d3-ease` for animation, `d3-time` for temporal
axes. It never calls `d3.select` or lets D3 touch the DOM. All rendering is JSX
producing `<svg>`, `<path>`, `<rect>`, `<circle>` elements.

**Containers and interaction.** Interactivity is layered via container
components that wrap the chart: `VictoryVoronoiContainer` (nearest-point
tooltips), `VictoryZoomContainer` (pan/zoom), `VictoryBrushContainer`
(range selection), `VictoryCursorContainer`, and `VictorySelectionContainer`.
Because a chart takes one container, combining behaviors requires
`createContainer("zoom", "voronoi")`, a factory that composes two container
behaviors into one[^5].

**Events and animation.** Every primitive accepts an `events` prop describing
target/eventHandlers/mutation triples; `VictorySharedEvents` propagates events
across sibling components (hovering a legend item highlighting a line). Animation
is declarative via the `animate` prop, backed by `victory-animation`'s
`d3-interpolate` transitions, including enter/exit transitions when data changes.

**Packaging.** The `victory` meta-package re-exports per-chart packages
(`victory-bar`, `victory-line`, `victory-axis`, `victory-core`, ...). You can
depend on the individual packages directly to narrow what you pull in. React
Native support lives in `victory-native`, which shares nearly all logic and
swaps the SVG renderer for `react-native-svg`.

## Production Notes

**SVG does not scale to large datasets.** This is the single most important
operational fact. Each datum is one or more DOM nodes; a scatter plot of
10,000+ points produces 10,000+ SVG elements, and initial render plus every
interaction becomes janky. There is no canvas or WebGL fallback in the core
library. For high-cardinality data, downsample/aggregate before plotting, or
choose a canvas-based library.

**Bundle size and tree-shaking.** Importing from the `victory` meta-package
tends to pull in more than a single chart type needs, and tree-shaking through
the re-export layer is imperfect. Import the specific package
(`import { VictoryBar } from "victory-bar"`) when bundle weight matters. The
D3 module dependencies also contribute measurable weight.

**Victory Native is two different things.** The `victory-native` package in this
repo is the SVG-based (`react-native-svg`) port that mirrors the web API.
Separately, `victory-native-xl` is a ground-up rewrite built on React Native
Skia and Reanimated with a different, hook-based API and much better performance
on device[^6]. They share the brand, not the code — do not assume web examples
translate to XL, and pick deliberately which one you are adopting.

**Responsive sizing.** Charts default to a fixed `width`/`height`. Fluid layouts
need `containerComponent={<VictoryContainer responsive />}` (the default) plus a
sized parent, or manual width measurement; naive percentage widths on the SVG do
not behave as many expect.

**Theming.** `VictoryTheme` (`material`, `grayscale`, `clean`) sets defaults, but
overrides are per-component `style` props with a `data`/`labels` split, and the
cascade between theme, group, and component styles is a frequent source of "why
didn't my color apply" confusion.

**Upgrades.** As of `victory@34.0.0` the minimum React is 16.3[^4]. Recent major
versions have modernized the TypeScript types and build; type definitions on
older releases were weaker, so pin and test types when upgrading a large chart
surface.

## When to Use / When Not

**Use when:**
- You want charts that behave like normal React components (props, events,
  theming) and integrate with your existing component model.
- Data volumes are small to moderate (hundreds to low thousands of points).
- You need the same API across web and React Native for standard chart types.
- You want fine control over individual marks, labels, and interactions.

**Avoid when:**
- You render tens of thousands of points or need 60fps on huge datasets — use a
  canvas/WebGL library.
- Bundle size is tightly constrained and you only need one simple chart.
- You want a batteries-included dashboard kit with minimal composition — a
  higher-level library will get you there faster.

## Alternatives

- recharts/recharts — also React + D3 + SVG and composable; more batteries
  included and easier defaults, less low-level control. Use when you want the
  same model with less assembly.
- plouc/nivo — React + D3 with SVG and canvas renderers plus server-side
  rendering; use when you need many chart types or a canvas escape hatch.
- airbnb/visx — thin React wrappers over raw D3 primitives; use when you want to
  build bespoke visualizations and accept doing the layout yourself.
- apache/echarts — canvas-based, framework-agnostic; use when dataset size or
  raw rendering performance is the priority over React-native composition.
- FormidableLabs/victory-native-xl — Skia-based React Native rewrite; use for
  performant mobile charts instead of the SVG `victory-native`.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-07 | Repository created; first Victory components published[^1]. |
| 34.0.0 | — | Minimum React raised to 16.3[^4]. |
| — | 2023 | Formidable absorbed into NearForm; docs move to commerce.nearform.com[^3]. |
| 37.x | 2025 | Current release line; ongoing maintenance releases[^4]. |

## References

[^1]: FormidableLabs/victory repository, created 2015-07-02. https://github.com/FormidableLabs/victory
[^2]: Victory documentation — components are composed in React and rendered as SVG, using D3 modules for scales and shapes. https://commerce.nearform.com/open-source/victory/docs
[^3]: Victory documentation and homepage hosted under NearForm's commerce open-source site. https://commerce.nearform.com/open-source/victory/
[^4]: Victory README — "As of victory@34.0.0 Victory requires React version 16.3.0 or above"; current published versions on npm. https://www.npmjs.com/package/victory
[^5]: Victory `createContainer` — combine container behaviors. https://commerce.nearform.com/open-source/victory/docs/victory-create-container
[^6]: Victory Native XL — Skia + Reanimated rewrite, separate API. https://github.com/FormidableLabs/victory-native-xl

## Tags

react, data-visualization, charts, svg, d3, typescript, react-native, components, frontend, charting-library
