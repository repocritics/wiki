# airbnb/visx

> Low-level React visualization primitives that wrap d3's math while letting React own the DOM.

[GitHub repo](https://github.com/airbnb/visx) ·
[Official website](https://visx.airbnb.tech) ·
[License: MIT](https://github.com/airbnb/visx/blob/master/LICENSE)

## Overview

visx (formerly `vx`) is a collection of unopinionated, low-level visualization
components for React, maintained by Airbnb[^1]. It is not a charting library in
the drop-in sense: there is no `<LineChart data={...} />` that renders a finished
graph. Instead visx gives you the pieces — scales, shapes, axes, gridlines,
tooltips, gradients, glyphs — and expects you to assemble them into an SVG
yourself. The design goal is that teams build their own reusable chart library on
top of visx rather than fighting a pre-baked one[^2].

The defining decision is the split of responsibilities: **d3 does the math,
React does the DOM.** visx wraps d3 modules (`d3-scale`, `d3-shape`, `d3-array`,
`d3-hierarchy`, and friends) for layout and geometry, but never touches the DOM
the way idiomatic d3 does — no `d3.select()`, no `enter()`/`exit()`/`update()`
joins, no copy-pasting d3 into `useEffect`. Everything renders as ordinary React
elements (mostly SVG), so component state, reconciliation, and SSR work exactly
as they do in any React app.

The tradeoff is verbosity. A basic bar chart is dozens of lines of explicit
scale setup, accessor functions, and element mapping. In exchange you get full
control over every rendered node and no framework opinions to escape. visx is
aimed at teams that will render many custom or brand-specific charts and want a
shared primitive layer, not at someone who needs one quick chart.

## Getting Started

visx is published as roughly 30 independent packages under the `@visx/` scope;
you install only the ones you use to keep bundles small[^2]. v4, the current
stable line, requires React 18 or 19[^3].

```bash
npm install @visx/scale @visx/shape @visx/group @visx/axis @visx/mock-data
```

```tsx
import { letterFrequency } from '@visx/mock-data';
import { scaleBand, scaleLinear } from '@visx/scale';
import { Group } from '@visx/group';
import { Bar } from '@visx/shape';

const data = letterFrequency;
const width = 500, height = 300;

const x = (d: typeof data[number]) => d.letter;
const y = (d: typeof data[number]) => d.frequency * 100;

const xScale = scaleBand({ range: [0, width], domain: data.map(x), padding: 0.4 });
const yScale = scaleLinear({ range: [height, 0], domain: [0, Math.max(...data.map(y))] });

export function BarChart() {
  return (
    <svg width={width} height={height}>
      <Group>
        {data.map((d) => {
          const barHeight = height - (yScale(y(d)) ?? 0);
          return (
            <Bar
              key={x(d)}
              x={xScale(x(d))}
              y={height - barHeight}
              width={xScale.bandwidth()}
              height={barHeight}
              fill="#fc2e1c"
            />
          );
        })}
      </Group>
    </svg>
  );
}
```

## Architecture / How It Works

visx is a **monorepo of small, single-purpose packages**, not one bundle. The
split matters because it is the whole value proposition — you take `@visx/scale`
and `@visx/shape` without pulling in geo projections or hierarchy layouts. Rough
groupings:

- **Math / marks**: `@visx/scale`, `@visx/curve`, `@visx/shape` (Bar, Line,
  LinePath, Arc, Pie, AreaClosed…), `@visx/glyph`, `@visx/text`.
- **Chart furniture**: `@visx/axis`, `@visx/grid`, `@visx/group`,
  `@visx/legend`, `@visx/annotation`.
- **Layouts (d3 wrappers)**: `@visx/hierarchy` (tree/cluster/treemap/pack),
  `@visx/network`, `@visx/geo`, `@visx/heatmap`, `@visx/stats`.
- **Interaction**: `@visx/tooltip`, `@visx/event`, `@visx/zoom`, `@visx/brush`,
  `@visx/drag`, `@visx/voronoi`, `@visx/responsive` (sizing hooks).
- **Higher-level**: `@visx/xychart` — an opinionated, composable API for common
  cartesian charts, added later for those who want less boilerplate.

Almost everything renders **SVG**. This is the source of both visx's ergonomics
(inspectable, stylable, accessible, animatable with any React animation library)
and its ceiling (SVG DOM nodes are expensive in the thousands).

**Animation is deliberately not included.** This is a repeatedly-litigated design
choice[^4]: baking in a motion library would bloat every consumer, and React
already has `react-spring`, `react-move`, `framer-motion`, etc. visx components
are plain React, so any of those compose on top. The cost is that smooth
transitions are your problem to wire up.

Because output is just React elements, server-side rendering and static
generation work with no special handling — a visx chart is a pure function of its
props.

## Production Notes

**Boilerplate is the real cost.** Expect to hand-write scales, accessors,
margins, and axis wiring for every chart unless you build an abstraction layer
(which is the intended usage). Teams that skip building that layer end up with
large amounts of near-duplicate chart code. `@visx/xychart` reduces this for
standard cartesian cases; reach for it before rolling your own if your charts are
conventional.

**SVG scaling limits.** Rendering thousands of individual marks (dense
scatterplots, high-frequency time series, large heatmaps) creates thousands of
DOM nodes and will jank on interaction. visx has no built-in canvas/WebGL
renderer. For large-N visuals, downsample, aggregate, or render to a `<canvas>`
yourself using visx scales for the math — a common escape hatch, since scales are
decoupled from rendering.

**Tooltips and coordinates.** `@visx/tooltip` gives you the state/portal
machinery but not placement magic; combine it with `@visx/event`'s `localPoint`
and often `@visx/voronoi` for nearest-point hit detection. This is more assembly
than higher-level libraries require.

**Responsive sizing.** Charts need explicit `width`/`height`. `@visx/responsive`
(`ParentSize`, `useScreenSize`) provides measured dimensions, but you must thread
them through — nothing is responsive by default.

**Upgrade friction is mild but real.** The `vx` → `visx` rename (scope change
from `@vx/*` to `@visx/*`) was a hard break for early adopters. The v3 → v4 jump
moved the peer dependency to React 18/19; a migration guide covers it[^3].
Because packages version together, a `@visx/*` upgrade usually means bumping the
whole set to matching majors. visx is TypeScript-first, and its d3-derived
generics can get verbose — accurate typing rather than a defect, but expect to
annotate accessors.

## When to Use / When Not

**Use when:**
- You will render many custom or brand-specific charts and want a shared
  primitive layer your team controls.
- You need full control over every rendered element (bespoke interactions,
  precise styling, non-standard chart forms).
- You want d3's math without d3's imperative DOM model, inside React.
- SSR / static rendering of charts matters.

**Avoid when:**
- You need one or two conventional charts fast — recharts or nivo will be a
  fraction of the code.
- Your dataset is very large (tens of thousands of points) and you are not
  prepared to drop to canvas/WebGL yourself.
- You want animation, theming, and tooltips working out of the box with no
  assembly.
- Your team won't invest in building the abstraction layer visx assumes.

## Alternatives

- recharts/recharts — declarative, batteries-included React charts; use it when you want standard charts with minimal code and don't need low-level control.
- plouc/nivo — React + d3 with SVG, Canvas, and HTML renderers plus theming; use it when you want visx-level breadth but pre-built, themeable chart components.
- FormidableLabs/victory — opinionated React (and React Native) chart components; use it when you need shared web/native charting.
- d3/d3 — the raw imperative toolkit visx wraps; use it when you want maximum control and don't need React to own the DOM.
- apache/echarts — canvas-based, framework-agnostic, huge feature set; use it for large datasets and dashboards where render performance dominates.

## History

| Version | Date | Notes |
|---------|------|-------|
| vx 0.0.x | 2017 | Original release as `vx` by Harrison Shoff[^1]. |
| 1.0 | 2020 | Rebranded to visx, moved under the Airbnb org, `@visx/*` scope[^1]. |
| 2.0 | 2021 | Package-wide major; d3 dependency updates. |
| 3.0 | 2023 | Major line preceding the React 18/19 requirement. |
| 4.0 | 2025 | Requires React 18 or 19; v3→v4 migration guide[^3]. |

## References

[^1]: Airbnb Engineering, "Introducing visx" — the rebrand of `vx` (Harrison Shoff's original library) into Airbnb's visx. https://medium.com/airbnb-engineering/introducing-visx-from-airbnb-fd6155ac4658
[^2]: visx README, "Motivation" — unopinionated, per-package installs, build-your-own-chart-library goal. https://github.com/airbnb/visx
[^3]: visx v4 migration guide and README note requiring React 18/19. https://github.com/airbnb/visx/blob/master/MIGRATION.md
[^4]: visx issue #6 — rationale for not baking in animation. https://github.com/airbnb/visx/issues/6

## Tags

typescript, react, data-visualization, d3, svg, charts, low-level, components, dataviz, frontend
