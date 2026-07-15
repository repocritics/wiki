# livebook-dev/livebook

> Interactive, collaborative Elixir notebooks — reproducible by construction, with code cells running on live BEAM nodes.

[GitHub repo](https://github.com/livebook-dev/livebook) ·
[Official website](https://livebook.dev) ·
[License: Apache-2.0](https://github.com/livebook-dev/livebook/blob/main/LICENSE)

## Overview

Livebook is a web application for writing interactive, collaborative code notebooks in Elixir. It was created in 2021 by José Valim (creator of Elixir) and the Dashbit team, and is the notebook counterpart to Elixir's numerical/data stack (Nx, Explorer, Bumblebee, Kino)[^1]. Where Jupyter treats the kernel as an opaque process reachable over a JSON protocol, Livebook is itself an Elixir application: a notebook's runtime is a real BEAM node, and cells are evaluated in-process with full access to the language, OTP, and the distribution layer.

The defining design commitment is **reproducibility**. Notebooks are stored as `.livemd` files — a subset of Markdown, so they diff and review cleanly in Git — and dependencies are declared inline via `Mix.install/2`, pinning exact versions the way a `mix.exs` would. Livebook tracks a dependency graph between cells and marks downstream cells "stale" when an upstream cell changes, so evaluation order is deterministic rather than whatever-you-happened-to-run-last. This is the direct answer to Jupyter's most cited failure mode: hidden state from out-of-order execution.

The tension in adopting Livebook is that all of this is Elixir-first. It is not a polyglot notebook: the interactive story, the Smart cells, the widget library (Kino), and the runtime model are built around the BEAM. There is Python support (cells and integration), but it runs alongside the Elixir runtime rather than replacing it[^2]. If your team is not already in — or willing to enter — the Elixir ecosystem, most of Livebook's leverage does not apply.

## Getting Started

The fastest path is the Docker image (no Elixir install required):

```shell
docker run -p 8080:8080 -p 8081:8081 --pull always \
  -u $(id -u):$(id -g) -v $(pwd):/data \
  ghcr.io/livebook-dev/livebook
```

Or run it directly with Elixir v1.18+ via the escript:

```shell
mix escript.install hex livebook
livebook server
```

A cell installing deps and rendering an interactive chart:

```elixir
Mix.install([
  {:kino_vega_lite, "~> 0.1"}
])

data = for i <- 1..100, do: %{x: i, y: :math.sin(i / 5)}

VegaLite.new(width: 600, height: 300)
|> VegaLite.data_from_values(data)
|> VegaLite.mark(:line)
|> VegaLite.encode_field(:x, "x", type: :quantitative)
|> VegaLite.encode_field(:y, "y", type: :quantitative)
|> Kino.VegaLite.new()
```

## Architecture / How It Works

Livebook is a Phoenix LiveView application, and its architecture separates the **editor** from the **runtime**:

- **The Livebook server** hosts the UI over LiveView and holds notebook state. Collaboration is real-time: multiple users edit the same notebook with no extra setup, coordinated through a CRDT so concurrent text edits converge without a central lock.
- **The runtime** is a separate Elixir node that actually evaluates code. By default Livebook spawns a fresh standalone node per notebook, but you can attach to a running node or run "embedded" inside an existing Mix project — which makes Livebook a live introspection tool for a real application, with access to all its modules and deps[^3].

Cells communicate results back through **Kino**[^4], the widget library. A Kino value is a process; rendering a table, chart, map, or input creates a live component that streams updates to the browser over the LiveView channel. This is why Livebook's outputs can be interactive (frames that update, inputs that feed later cells) rather than static images.

**Smart cells** are the higher-level authoring surface: a UI form (database query, chart builder, file upload, map) that generates the underlying Elixir code, which you can inspect and edit. They are implemented as Kino components and are extensible — a library can ship its own Smart cell.

**Reproducibility** is enforced by two mechanisms working together: `Mix.install/2` pins the dependency set at the top of the notebook, and the cell dependency tracker computes which cells are stale after any edit. The `.livemd` format persists the notebook (including Mermaid diagrams and KaTeX math) as reviewable Markdown, so the notebook, its deps, and its execution semantics all live in version control.

## Production Notes

**Livebook executes arbitrary code — treat every instance as a shell.** Anyone who can reach a Livebook can read any file and run any command as the Livebook user. By default it binds to `127.0.0.1` and, in production, generates a token that must be present in the URL. Exposing it to a network without `LIVEBOOK_PASSWORD` (min 12 chars) or an identity provider is equivalent to publishing an unauthenticated remote shell[^5].

**Deployment surface is large.** Livebook is configured almost entirely through `LIVEBOOK_*` environment variables — dozens of them, covering ports, base URL paths (for reverse proxies), clustering, proxy headers, and Zero Trust auth (`LIVEBOOK_IDENTITY_PROVIDER` supports Cloudflare Access, Google IAP, Tailscale, basic auth). Behind a proxy, `LIVEBOOK_BASE_URL_PATH` and `LIVEBOOK_PROXY_HEADERS` usually both need setting, and the separate iframe port (8081) for untrusted output must be reachable.

**Apps mode changes the threat model.** A notebook can be deployed as a multi-session "app" via `LIVEBOOK_APPS_PATH`. This is useful for turning a notebook into a small internal tool, but each app still runs Elixir code; password handling (`LIVEBOOK_APPS_PATH_PASSWORD`) and warmup (`LIVEBOOK_APPS_PATH_WARMUP`, relevant for Docker cold-start latency) need explicit configuration.

**It is a CLI/standalone tool, not a library.** The README is explicit: the `livebook` Hex package is not supported as a Mix/Hex dependency — do not add it to a project's `mix.exs`. Embedding Livebook means running the server and attaching to your app's node, not linking it.

**Clustering has sharp edges.** Running multiple instances requires `LIVEBOOK_SECRET_KEY_BASE`, a shared `LIVEBOOK_COOKIE`, FQDN/IP node names (long-name distribution only), and a `LIVEBOOK_CLUSTER` strategy. This is standard BEAM-distribution operational knowledge; teams without Erlang cluster experience should expect a learning curve.

## When to Use / When Not

**Use when:**
- Your team works in Elixir and wants reproducible, Git-reviewable notebooks over Jupyter's hidden-state model.
- You want live introspection of a running Elixir/Phoenix app (embedded runtime).
- You need real-time collaborative editing without standing up extra infrastructure.
- You're doing Elixir-native data/ML work (Nx, Explorer, Bumblebee) and want Kino's interactive outputs.

**Avoid when:**
- Your notebooks are primarily Python/R/Julia — the ecosystem and Smart cells are Elixir-centric; use a native tool.
- You need a notebook you can embed as a plain library dependency.
- You want a managed, hardened multi-tenant SaaS notebook out of the box (self-hosting security is on you, or use Livebook Teams).

## Alternatives

- jupyter/notebook — the incumbent; polyglot via kernels, huge ecosystem. Use instead when your stack is Python/R/Julia rather than Elixir.
- marimo-team/marimo — reactive, reproducible Python notebooks stored as `.py`; shares Livebook's anti-hidden-state philosophy. Use when you want the same guarantees but in Python.
- fonsp/Pluto.jl — reactive notebooks for Julia with automatic dependency re-runs. Use for Julia numerical work.
- apache/zeppelin — JVM/Spark-oriented multi-language notebooks. Use for big-data pipelines on the JVM.
- Observable — reactive JavaScript notebooks for data visualization on the web. Use for browser-native, JS/D3 dataviz.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2021-01 | Repository created at Dashbit[^1]. |
| 0.1 | 2021 | First public release; Phoenix LiveView editor + `.livemd` format[^1]. |
| 0.x | 2022 | Kino widget library and Smart cells for code-generating UI[^4]. |
| 0.x | 2023 | Apps deployment (`LIVEBOOK_APPS_PATH`) and Livebook Teams introduced[^6]. |
| 0.x | 2026 | Actively developed (main branch pushed 2026-07); requires Elixir v1.18+[^7]. |

## References

[^1]: Livebook — official site and announcement history. https://livebook.dev/
[^2]: Livebook documentation — Python and multi-language support. https://hexdocs.pm/livebook/
[^3]: Livebook README — custom runtimes (standalone, attached, embedded). https://github.com/livebook-dev/livebook#readme
[^4]: Kino — Livebook's interactive widget library. https://github.com/livebook-dev/kino
[^5]: Livebook README — "Security considerations". https://github.com/livebook-dev/livebook#readme
[^6]: Livebook Teams — commercial deployment offering by Dashbit. https://livebook.dev/teams/
[^7]: livebook-dev/livebook GitHub API metadata, retrieved 2026-07-15.

## Tags

elixir, notebooks, phoenix-liveview, data-science, reproducible-research, collaborative, interactive-computing, beam, machine-learning, dataviz
