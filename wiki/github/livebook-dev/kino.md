# livebook-dev/kino

> The rendering and widget layer of Livebook — the library that turns Elixir
> values into interactive notebook output.

[GitHub repo](https://github.com/livebook-dev/kino) ·
[Docs](https://hexdocs.pm/kino) ·
[License: Apache-2.0](https://github.com/livebook-dev/kino/blob/main/LICENSE)

## Overview

Kino is the client-side library that Livebook notebooks use to render rich and
interactive output: data tables, inputs, buttons, frames that update in place,
custom JavaScript widgets, and "smart cells" (UI-driven cells that generate
Elixir code)[^1]. It was created by Dashbit in mid-2021, a few months after
Livebook itself, and is maintained by the Livebook core team; the repo
originally lived under the `elixir-nx` organization before moving to
`livebook-dev`.

Its star count (448 as of mid-2026) wildly understates its footprint. Kino is
infrastructure consumed indirectly: nearly every non-trivial Livebook notebook
starts with `Mix.install([{:kino, ...}])`, and the official integration
packages (`kino_vega_lite`, `kino_explorer`, `kino_db`, `kino_bumblebee`) all
build on its extension APIs[^2]. Development is active (pushed June 2026) with
a small, well-triaged issue surface (7 open issues).

The defining tradeoff: Kino output is meaningless outside a Livebook runtime —
`Kino.render/1` in a Mix project or IEx shell does nothing useful. In exchange
for that coupling you get a widget model that is genuinely process-based:
widgets are supervised Elixir processes that push updates to the browser, not
static HTML snapshots.

## Getting Started

Kino is installed inside a Livebook notebook's setup cell, not as a Mix
dependency of an application:

```elixir
Mix.install([
  {:kino, "~> 0.19.0"}
])
```

Rendering a table and wiring a button to a live-updating frame:

```elixir
data = [
  %{name: "Ada", language: "Elixir"},
  %{name: "Grace", language: "COBOL"}
]

Kino.DataTable.new(data)
```

```elixir
frame = Kino.Frame.new() |> Kino.render()
button = Kino.Control.button("Ping") |> Kino.render()

Kino.listen(button, fn _event ->
  Kino.Frame.render(frame, Kino.Markdown.new("pong at #{DateTime.utc_now()}"))
end)
```

## Architecture / How It Works

Kino runs inside the notebook's runtime node and talks to the Livebook server
over the runtime's existing communication channel — outputs travel alongside
standard IO, relayed through the evaluation process's group leader. The key
pieces:

- **`Kino.Render` protocol** — decides how a cell's result value becomes an
  output. Libraries implement it for their structs (Explorer dataframes,
  VegaLite specs) so plain values render richly without explicit calls.
- **`Kino.JS`** — the custom-widget API. A widget module embeds its JavaScript
  assets (compiled into the Hex package at build time); Livebook serves them
  to the browser and the JS side receives the widget's data via a small
  runtime API[^3].
- **`Kino.JS.Live`** — stateful widgets backed by a GenServer-like behaviour
  (`init`, `handle_connect`, `handle_event`, broadcasts). Every browser client
  connected to the notebook gets a connection; events flow both directions.
  This is the mechanism behind tables with pagination, live charts, and forms.
- **`Kino.SmartCell`** — extends `Kino.JS.Live` with a `to_source/1` callback.
  A smart cell is a UI that generates real, inspectable Elixir code into the
  notebook — Livebook's answer to "low-code without lock-in"[^4]. Database
  connection cells and chart builders in the official packages are smart
  cells.
- **Process lifecycle** — widgets started during a cell evaluation are
  terminated when that cell is reevaluated. `Kino.start_child/1` puts
  long-lived processes under a supervisor owned by the notebook runtime
  instead, surviving reevaluation of other cells.

The core package is deliberately lean; anything with heavyweight dependencies
(VegaLite, Explorer, DB drivers) lives in a separate `kino_*` package that
depends on kino's extension APIs. That split happened in 2022 and is why the
core has stayed small.

## Production Notes

- **Livebook version coupling.** Kino is pre-1.0 and evolves in lockstep with
  Livebook; each Livebook release supports a range of Kino versions, and
  mismatches can produce outputs that silently fail to render. Pin kino in
  `Mix.install` and expect to bump it when upgrading Livebook.
- **`Kino.Input.read/1` reads at evaluation time.** Changing an input does not
  re-run anything; Livebook marks dependent cells stale and the user must
  reevaluate. Code assuming reactive re-execution (Shiny/marimo style) will
  surprise you.
- **Memory lives in processes.** `Kino.DataTable.new/1` holds the full dataset
  in the widget process. For large data, use `kino_explorer` with lazy
  dataframes rather than materializing lists of maps.
- **Reevaluation kills widget state.** Any widget not started via
  `Kino.start_child/1` dies on cell reevaluation. Background loops feeding a
  chart must be structured around this or they silently stop.
- **Offline/airgapped deploys.** Official widget assets are bundled into the
  Hex packages, so they work without internet. Custom `Kino.JS` widgets that
  load from CDNs break in airgapped Livebook deployments — bundle everything.
- **Livebook apps.** When a notebook is deployed as a Livebook app, Kino
  inputs and controls become the app's entire UI, and each user session gets
  its own runtime — test multi-user behavior before deploying.

## When to Use / When Not

**Use when:**
- You are writing Livebook notebooks and want tables, charts, forms, or
  live-updating output — Kino is the only game in town, and a good one.
- You maintain an Elixir library and want your structs to render well in
  notebooks (implement `Kino.Render`, ship a `kino_*` companion package).
- You want to prototype an internal tool and deploy it as a Livebook app
  without writing a Phoenix frontend.

**Avoid when:**
- You need visualization in a regular Elixir application — use Phoenix
  LiveView, or generate VegaLite/plotly specs directly.
- You expect reactive dataflow notebooks (marimo, Pluto.jl style); Livebook's
  evaluation model is explicit and Kino inherits it.
- You need a stable 1.x API contract — Kino is 0.x and has had breaking
  changes across minor versions.

## Alternatives

- jupyter-widgets/ipywidgets — the equivalent layer for Jupyter; use it when
  your stack is Python, not Elixir.
- marimo-team/marimo — reactive Python notebooks with built-in UI elements;
  use it when you want dataflow re-execution rather than explicit reevaluation.
- fonsp/Pluto.jl — reactive Julia notebooks with bind-able inputs; the Julia
  ecosystem's counterpart.
- streamlit/streamlit — use it when the goal is a shareable data app first and
  a notebook never; Livebook apps + Kino cover part of this ground in Elixir.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021-06 | Initial release; VegaLite, ETS, and data table widgets shipped in core[^2]. |
| 0.5.x | 2022-01 | `Kino.JS` / `Kino.JS.Live` — public custom-widget API (Livebook 0.5 era)[^3]. |
| 0.6.x | 2022-04 | `Kino.SmartCell`; chart/table integrations extracted to `kino_vega_lite` and sibling packages[^4]. |
| 0.19.x | 2026 | Current series; README pins `~> 0.19.0`. Full log in CHANGELOG[^2]. |

## References

[^1]: Kino documentation on HexDocs. https://hexdocs.pm/kino
[^2]: Kino changelog. https://github.com/livebook-dev/kino/blob/main/CHANGELOG.md
[^3]: `Kino.JS` module docs. https://hexdocs.pm/kino/Kino.JS.html
[^4]: `Kino.SmartCell` module docs. https://hexdocs.pm/kino/Kino.SmartCell.html

## Tags

elixir, livebook, notebooks, interactive-widgets, data-visualization, charts, vega-lite, smart-cells, beam, developer-tools
