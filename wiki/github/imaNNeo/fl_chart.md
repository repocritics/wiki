# imaNNeo/fl_chart

> A pure-Dart Flutter charting library that paints every chart type onto Flutter's own canvas, trading raw scalability for deep visual control.

[GitHub repo](https://github.com/imaNNeo/fl_chart) ·
[Official website](https://flchart.dev) ·
[License: MIT](https://github.com/imaNNeo/fl_chart/blob/main/LICENSE)

## Overview

fl_chart is the most-used third-party charting package in the Flutter
ecosystem, with ~7.6k GitHub stars and a large presence on pub.dev. It is a
single-author project (Iman Khoshabi) that has been developed since 2019 and
funded through GitHub Sponsors and donations rather than a company[^1]. It
covers six chart families out of the box: line, bar, pie, scatter, radar, and
candlestick.

The library's defining choice is that it draws everything with Flutter's own
`CustomPainter` on a `Canvas` — there is no HTML, no WebView, no platform chart
SDK underneath. That gives it two properties that shape every tradeoff below:
you get pixel-level control over how a chart looks (gradients, dashed lines,
custom tooltips, arbitrary touch regions), and you pay for it by re-painting the
full chart on Flutter's UI thread. There is no data virtualization or
decimation layer. For dashboards with tens to low-hundreds of points per series
this is a non-issue; for real-time streams or datasets of thousands of points
it becomes the primary thing you tune around.

The API is verbose-but-honest: charts are configured through deeply nested
immutable data classes (`LineChartData`, `BarChartData`, …) with `copyWith`,
and animation is achieved by handing the widget a new data object that it
interpolates toward. Historically the API changed often — fl_chart spent about
six years in `0.x`, and minor bumps frequently carried breaking renames — so
much online example code targets an API that no longer compiles[^2].

## Getting Started

```bash
flutter pub add fl_chart
```

```dart
import 'package:fl_chart/fl_chart.dart';
import 'package:flutter/material.dart';

class SalesChart extends StatelessWidget {
  const SalesChart({super.key});

  @override
  Widget build(BuildContext context) {
    return LineChart(
      LineChartData(
        lineBarsData: [
          LineChartBarData(
            spots: const [
              FlSpot(0, 3), FlSpot(1, 1), FlSpot(2, 4),
              FlSpot(3, 2), FlSpot(4, 5),
            ],
            isCurved: true,
            barWidth: 3,
            dotData: const FlDotData(show: false),
          ),
        ],
        titlesData: const FlTitlesData(show: true),
        gridData: const FlGridData(show: true),
        borderData: FlBorderData(show: false),
      ),
      // Passing a *new* LineChartData rebuilds and animates toward it.
      duration: const Duration(milliseconds: 250),
      curve: Curves.easeInOut,
    );
  }
}
```

## Architecture / How It Works

Each chart family is a pair: a public `ImplicitlyAnimatedWidget`-style widget
(`LineChart`, `BarChart`, …) and a private `CustomPainter` that renders it. The
widget holds a `*ChartData` value object; when you rebuild with a new data
object, fl_chart `lerp`s between the old and new data every frame and hands the
interpolated result to the painter. Every `*ChartData` subtree implements
`lerp`, which is why animation "just works" but also why the data model is so
large — interpolation has to be defined for every visual property.

Rendering is a full repaint. On each frame the painter walks the data, maps
data-space coordinates into pixel-space, and issues `Canvas` draw calls (paths,
rects, arcs, text). There is no retained scene graph or dirty-region tracking of
individual series; a change anywhere re-paints the chart. Touch handling is
layered on top: charts expose `*TouchData` config with callbacks that receive a
hit-test response (which bar/spot/section was touched), letting you drive
tooltips and selection from your own state.

Because output is raw canvas pixels, fl_chart inherits Flutter's renderer
characteristics directly — it renders identically across mobile, web, and
desktop, and its performance and text rasterization track whichever engine
backend is active (Impeller or the older Skia/CanvasKit path). It also inherits
the downside: canvas drawings carry no semantics, so charts are effectively
invisible to screen readers unless you wrap them in your own `Semantics`.

## Production Notes

- **Large / streaming datasets are the main footgun.** There is no built-in
  downsampling. Thousands of `FlSpot`s, or high-frequency `setState` updates,
  will drop frames because each update triggers a full repaint on the UI thread.
  Standard mitigations: decimate data before handing it to the chart, cap the
  visible window, and disable per-point `dotData` on dense line series.
- **Deeply nested `copyWith` gets painful.** Updating one property several
  levels down (e.g. a single axis title style) means threading `copyWith`
  through each enclosing data object. Teams commonly wrap chart construction in
  helper builders to keep call sites readable.
- **Upgrades bite.** Across the long `0.x` run, minor versions renamed and
  restructured APIs (for example, animation parameters and titles/axis config
  were reworked more than once). Pin the version, read the changelog before
  bumping, and distrust old tutorials — a lot of copy-pasted example code
  targets removed APIs[^2].
- **Accessibility is on you.** No semantic labels are emitted by default; add
  `Semantics` wrappers or an alternative data table if screen-reader support
  matters.
- **1.0.0 (May 2025) is the stability line.** Before it, treat any given
  release as potentially breaking; after it, the project follows semantic
  versioning more predictably[^3].

## When to Use / When Not

**Use when:**
- You want native, cross-platform Flutter charts with no WebView and no
  per-platform native SDK.
- Visual customization matters — bespoke gradients, tooltips, touch regions,
  animated transitions between states.
- Your series are small-to-moderate (dashboards, forms, app metrics).

**Avoid when:**
- You need to render thousands of points or high-rate real-time streams without
  building your own decimation/windowing layer.
- You need built-in accessibility/semantics out of the box.
- You need chart types fl_chart doesn't cover (e.g. heatmaps, complex financial
  overlays, Gantt) — a heavier commercial suite may fit better.

## Alternatives

- google/charts — the old material `charts_flutter` package; broad but
  effectively unmaintained. Use only for legacy code, not new projects.
- entronad/graphic — grammar-of-graphics Flutter viz; use when you want a
  declarative, layered chart specification instead of imperative data objects.
- syncfusion_flutter_charts — commercially licensed, very wide chart-type
  coverage and support; use when you need financial/enterprise chart types and
  a vendor SLA, and can accept the license.
- flutter_echarts — wraps Apache ECharts in a WebView; use when you need
  ECharts' chart variety and are fine with WebView overhead and interop.
- A hand-rolled `CustomPainter` — use when your chart is simple and fixed, and
  a full library's data model is more ceremony than it's worth.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-05 | First public release; line/bar/pie on `CustomPainter`. |
| 0.55.0 | 2022-06 | Mid-`0.x` line; frequent breaking minor bumps through this era. |
| 0.68.0 | 2024-05 | Candlestick and later chart-type/config additions land in the 0.6x series. |
| 0.70.0 | 2024-12 | Final stretch of the `0.7x` pre-1.0 line. |
| 0.71.0 | 2025-04 | Last `0.x` release before the 1.0 cut. |
| 1.0.0 | 2025-05-08 | API-stability milestone; semantic versioning going forward[^3]. |
| 1.1.0 | 2025-08-31 | First minor after 1.0. |
| 1.2.0 | 2026-03-13 | Latest release at time of writing[^4]. |

## References

[^1]: fl_chart README and sponsors section — GitHub Sponsors / Buy Me A Coffee funding, single maintainer (Iman Khoshabi). https://github.com/imaNNeo/fl_chart
[^2]: fl_chart documentation index and migration notes; long `0.x` history with breaking minor releases. https://github.com/imaNNeo/fl_chart/blob/main/repo_files/documentations/index.md
[^3]: fl_chart 1.0.0 release, 2025-05-08 — pub.dev / GitHub releases. https://github.com/imaNNeo/fl_chart/releases/tag/1.0.0
[^4]: fl_chart 1.2.0 release, 2026-03-13. https://github.com/imaNNeo/fl_chart/releases/tag/1.2.0

## Tags

dart, flutter, charts, data-visualization, custom-painter, line-chart, bar-chart, pie-chart, candlestick, cross-platform, mit-license
