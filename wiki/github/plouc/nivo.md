# plouc/nivo

> React dataviz component library built on d3, with SVG, HTML, and Canvas renderers for the same charts.

[GitHub repo](https://github.com/plouc/nivo) ·
[Official website](https://nivo.rocks) ·
[License: MIT](https://github.com/plouc/nivo/blob/master/LICENSE.md)

## Overview

nivo is a collection of React charting components created by Raphaël Benitte (plouc) and first released in 2016[^1]. It wraps d3's math (scales, shapes, color, interpolation) but keeps the DOM under React's control rather than letting d3 mutate nodes directly — sidestepping the classic React/d3 ownership conflict. Charts are configured declaratively through props; there is no imperative chart object to manage.

The defining design choice is that most chart types ship in up to three rendering flavors: SVG (the default, crisp and fully interactive), HTML, and Canvas (for large datasets). This is nivo's main differentiator against simpler React chart libraries and its main source of surface-area complexity — the same chart has multiple component variants, each with slightly different capabilities.

nivo is a monorepo of scoped npm packages (`@nivo/core` plus one package per chart family). You install only the charts you use. Its documentation site at nivo.rocks is itself a selling point: an interactive playground where you tweak props live and it generates the corresponding code. With ~14k stars, active commits through mid-2026, and a broad chart catalog, it is one of the more complete React-native dataviz options, though it has famously never shipped a 1.0 — the whole project remains on 0.x versioning.

## Getting Started

```bash
npm install @nivo/core @nivo/bar
```

Every chart family is a separate package; `@nivo/core` holds the shared primitives (theming, patterns, container, motion config).

```tsx
import { ResponsiveBar } from '@nivo/bar'

const data = [
  { country: 'AD', hot: 42, cold: 18 },
  { country: 'AE', hot: 55, cold: 30 },
]

export default function Chart() {
  return (
    // ResponsiveBar fills its parent — the parent MUST have a real height
    <div style={{ height: 400 }}>
      <ResponsiveBar
        data={data}
        keys={['hot', 'cold']}
        indexBy="country"
        margin={{ top: 20, right: 20, bottom: 40, left: 40 }}
        padding={0.3}
        axisBottom={{ legend: 'country', legendOffset: 32 }}
      />
    </div>
  )
}
```

## Architecture / How It Works

**Monorepo of scoped packages.** `@nivo/bar`, `@nivo/line`, `@nivo/pie`, `@nivo/heatmap`, `@nivo/geo`, `@nivo/network`, `@nivo/sankey`, `@nivo/treemap`, `@nivo/calendar`, and many more are independent packages so an app pulls in only the chart code it needs. All of them depend on `@nivo/core`, which centralizes the theme context, SVG `defs` (patterns and gradients), the responsive container, and the animation config.

**Three render backends per chart.** A typical family exports `Bar` (SVG), `BarCanvas` (Canvas), and sometimes an HTML variant, each with a `Responsive*` wrapper. SVG gives one DOM node per mark — good for interactivity, CSS, and accessibility, bad for tens of thousands of elements. Canvas paints to a single element and scales to large datasets, at the cost of per-mark DOM (marks are hit-tested internally for tooltips, but you cannot style them with CSS or attach DOM event handlers).

**d3 for math, React for the DOM.** nivo uses d3-scale, d3-shape, d3-color and friends to compute geometry, then renders that geometry with React elements. d3 never touches the DOM. This keeps rendering inside React's reconciliation and lets server-side rendering work.

**Responsive wrappers.** `ResponsiveBar` and its siblings wrap the fixed-size base component in a measured container that reads its parent's dimensions (via a resize observer) and passes explicit `width`/`height` down. The base non-responsive components require you to pass `width` and `height` yourself.

**Animation via react-spring.** Transitions are driven by react-spring; the `animate` boolean and `motionConfig` prop control them globally per chart[^2]. nivo migrated to react-spring from react-motion earlier in its history.

**Theming.** A single theme object is threaded through context and covers axes, grid, labels, tooltips, and annotations. Patterns and gradients are declared as reusable `defs` and referenced by id from data or rules.

## Production Notes

**The zero-height container trap.** The single most common nivo bug: a `Responsive*` chart rendered inside a parent with no explicit height collapses to zero and shows nothing. Charts need a sized parent (fixed height, a flex/grid cell with resolved dimensions, or `100%` of a sized ancestor).

**SVG has a hard performance ceiling.** Hundreds to low thousands of SVG marks render fine; beyond that, interaction and re-render jank appears because each mark is a DOM node. The intended fix is switching to the `*Canvas` variant, but that is a different component with a reduced feature set (no DOM-level customization of marks, custom layers work differently). Plan the SVG-vs-Canvas decision up front — swapping late is a rewrite of that chart's customizations.

**Perpetual 0.x, real breaking changes.** nivo has never released 1.0, and minor bumps (e.g. 0.x → 0.x+1) have historically included breaking prop renames and theme-shape changes. There is no semver stability contract. Pin exact versions and read the release notes before upgrading; theme object reorganizations across versions are a recurring upgrade pain.

**Bundle size.** Importing only the chart packages you use is the main lever, but a single chart still transitively pulls in d3 modules and react-spring. Tree-shaking helps within a package; the package split is what keeps unused chart families out of the bundle. Canvas variants avoid large DOM but do not shrink the JS.

**Peer dependencies.** react-spring is a peer dependency, and version mismatches surface as runtime animation errors rather than install-time failures. Keep the react-spring range aligned with what your nivo version expects.

**TypeScript.** Types ship with the packages and are generally usable, but some prop unions are loose and a few areas fall back to `any`, so the compiler will not catch every misconfiguration — the runtime and the docs playground are still the fastest feedback loop.

**Server-side rendering.** SVG and HTML components render to markup on the server. A separate nivo HTTP API service exists to generate charts (SVG/PNG) server-side for non-React consumers, but it covers only a subset of chart types and is a heavier operational commitment than the client library.

## When to Use / When Not

**Use when:**
- You want ready-made, declarative React charts with deep prop-level customization.
- You need multiple chart families (bar, line, heatmap, geo, network, sankey, calendar) from one coherent, themeable system.
- You have large datasets and can use the Canvas variants for the heavy views.
- You value the live playground/code-generator workflow for iterating on chart config.

**Avoid when:**
- You want the smallest possible bundle for one or two simple charts — a lighter library or hand-rolled SVG wins.
- You need long-term API stability with rare breaking changes — 0.x versioning and prop churn work against you.
- Your dashboards are extremely large/complex and imperative — a non-React-native engine may scale further.
- You need charts outside React without standing up the HTTP API service.

## Alternatives

- recharts/recharts — composable React chart primitives, gentler learning curve, less customization depth; use when you want simple charts fast.
- airbnb/visx — low-level d3-as-React primitives with no chart opinions; use when you want to assemble bespoke charts yourself.
- apache/echarts — imperative, very large feature set, not React-native (needs a wrapper); use for big, complex, high-density dashboards.
- FormidableLabs/victory — declarative React charts that also target React Native; use when you need cross-platform mobile charts.
- d3/d3 — the raw toolkit nivo wraps; use directly when you need total control over rendering and interaction.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016 | First public release: SVG React components on top of d3[^1]. |
| 0.x | 2016–present | Never reached 1.0; chart catalog and Canvas/HTML variants grew across minor releases. |
| — | ~2020 | TypeScript migration and move to react-spring for transitions[^2]. |
| — | 2026-07 | Still actively committed to (last push 2026-07-11); ~14k stars, MIT. |

## References

[^1]: nivo repository and documentation. https://github.com/plouc/nivo — live at https://nivo.rocks
[^2]: react-spring, the animation library nivo uses for transitions. https://www.react-spring.dev/
[^3]: nivo components explorer and interactive playground. https://nivo.rocks/components/

## Tags

react, dataviz, charts, d3, svg, canvas, typescript, visualization, components, frontend
