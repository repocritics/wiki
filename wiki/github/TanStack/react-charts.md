# TanStack/react-charts

> A declarative React charting library with a small API surface, driven by D3 internally — now archived and unmaintained.

[GitHub repo](https://github.com/TanStack/react-charts) ·
[Official website](https://react-charts.tanstack.com) ·
[License: MIT](https://github.com/TanStack/react-charts/blob/beta/LICENSE)

## Overview

react-charts is a React charting library from Tanner Linsley, part of the TanStack family (React Query, React Table, React Form)[^1]. Its pitch was the opposite of the kitchen-sink charting libraries: a handful of primitives — line, bar, column, bubble, area — configured declaratively through a single `<Chart>` component and a memoized options object, with D3 doing scales and shape math invisibly underneath[^2]. It never aimed to be a dashboarding toolkit; it aimed to be the smallest thing that draws a good-looking, interactive, responsive chart in React.

The defining tension is maturity versus ambition. The v2 line was a hooks-and-TypeScript rewrite that shipped under `2.0.0-beta.*` versions and effectively lived in perpetual beta — the docs, types, and API were good enough to use in production, but a stable 2.0.0 final never landed. In March 2025 the repository was archived and the README was topped with a "No Longer Maintained" notice[^3]. What exists today is a frozen, MIT-licensed, beta-tagged library that still works with the React versions of its era but will receive no fixes, no React 19 validation, and no new chart types.

The repo also carries some naming archaeology: it began under `tannerlinsley/react-charts`, moved into the `TanStack` org, and GitHub still redirects the old path. The npm package remains `react-charts`.

## Getting Started

```bash
npm install react-charts@beta
# the stable dist-tag points at the old v1; v2 lives on the beta tag
```

```tsx
import { useMemo } from "react";
import { Chart } from "react-charts";

export default function MyChart() {
  const data = useMemo(
    () => [
      {
        label: "Series 1",
        data: [
          { primary: new Date("2024-01-01"), secondary: 10 },
          { primary: new Date("2024-02-01"), secondary: 20 },
          { primary: new Date("2024-03-01"), secondary: 15 },
        ],
      },
    ],
    []
  );

  const primaryAxis = useMemo(() => ({ getValue: (d) => d.primary }), []);
  const secondaryAxes = useMemo(
    () => [{ getValue: (d) => d.secondary, elementType: "line" }],
    []
  );

  return (
    <div style={{ width: "600px", height: "300px" }}>
      <Chart options={{ data, primaryAxis, secondaryAxes }} />
    </div>
  );
}
```

The parent element must have an explicit width and height — the chart fills its container and renders nothing in a zero-height box, which is the single most common "why is it blank" report.

## Architecture / How It Works

The public API is deliberately narrow: one `<Chart>` component and one `options` object. Inside that object, `data` is an array of series, `primaryAxis` and `secondaryAxes` are accessor-based axis configs, and per-series `elementType` (`"line" | "area" | "bar" | "bubble"`) selects the mark. Everything else — tooltips, cursors, focus behavior, stacking, inversion — is a flag or callback on the same object.

Underneath, D3 does the numeric work: `d3-scale` for axis scales, `d3-shape` for path generation, `d3-time`/`d3-array` for ticks and extents. React owns the DOM — the library emits SVG elements as React nodes rather than letting D3 mutate the DOM, so reconciliation and event handling stay inside React's model[^2]. Animations were handled by a spring-based motion layer (the repo's original `react-motion` topic reflects this lineage), which is why transitions are physics-driven rather than duration-based.

Two architectural consequences follow. First, **memoization is load-bearing, not optional**. Because the whole configuration is one object, passing a freshly-constructed `data`/`primaryAxis`/`secondaryAxes` on every render re-runs scale computation and can thrash or throw; the docs wrap each in `useMemo` for this reason. Second, the accessor pattern (`getValue`) means the library never assumes a data shape — you map your rows to primary/secondary values — which is flexible but pushes date/number coercion onto the caller.

## Production Notes

- **It is archived.** No security patches, no React 19 compatibility statement, no bug triage. Treat adoption as vendoring a frozen dependency; any breakage from a future React or bundler is yours to fix or fork.
- **Stable vs beta tag confusion.** `npm install react-charts` resolves to the old v1 API; the modern documented API is `react-charts@beta`. Installing the wrong one produces a component whose props don't match any current tutorial.
- **Zero-size container = blank chart.** The SVG sizes to its parent. A flex/grid parent that collapses to zero height renders nothing with no error.
- **Everything must be memoized.** Inline options objects cause avoidable re-computation and, in some configurations, runtime errors on rapid re-render. This is the top real-world footgun.
- **TypeScript types shipped but are beta-quality.** Generics over the data/datum types work for the common cases and get awkward around custom series and mixed `elementType`s; expect occasional `as` casts.
- **Feature ceiling is low by design.** No heatmaps, candlesticks, radial/pie, or geo. If a product requirement grows past the five built-in mark types, there is no escape hatch short of dropping to raw D3/SVG.

## When to Use / When Not

**Use when:**
- You want a few clean, responsive, interactive time-series or categorical charts and value a tiny API over configurability.
- You already accept the archived status and want a stable, dependency-light snapshot you control.
- You prefer accessor-based data mapping and React-owned SVG over imperative D3.

**Avoid when:**
- You are starting a new project in 2026 — pick a maintained library instead of adopting an archived one.
- You need chart types beyond line/bar/column/area/bubble, or heavy customization/theming.
- You want guaranteed React 19+ support, ongoing security fixes, or an active issue tracker.

## Alternatives

- recharts/recharts — the most-used maintained React charting library; composable component API, broader chart-type coverage. Use when you want an active project with a similar declarative feel.
- plouc/nivo — D3 + React with rich defaults, theming, and server-side rendering. Use when you want batteries-included visuals out of the box.
- airbnb/visx — low-level D3 primitives as React components. Use when you want to build bespoke charts and are willing to assemble them yourself.
- FormidableLabs/victory — declarative React chart components that also target React Native. Use when you need shared chart code across web and mobile.
- apache/echarts (via echarts-for-react) — canvas/SVG engine with very high feature coverage. Use when you need dense dashboards, large datasets, or exotic chart types.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-02 | Repo created under `tannerlinsley/react-charts`[^1]. |
| v1 | 2017–2019 | Original API; remains the default npm `latest` dist-tag. |
| v2 beta | ~2020 onward | TypeScript + hooks rewrite; single `options` object API. Shipped only as `2.0.0-beta.*`; never a stable 2.0.0[^2]. |
| moved to TanStack | — | Repo relocated into the TanStack org; old path redirects. |
| archived | 2025-03 | "No Longer Maintained" notice added; repository archived[^3]. |

## References

[^1]: TanStack/react-charts repository and README. https://github.com/TanStack/react-charts
[^2]: react-charts documentation — API, guides, and examples. https://react-charts.tanstack.com
[^3]: README "No Longer Maintained" notice; repository archived (last push 2025-03-10, per GitHub API). https://github.com/TanStack/react-charts

## Tags

react, charts, data-visualization, d3, svg, typescript, frontend, archived, tanstack, charting-library
