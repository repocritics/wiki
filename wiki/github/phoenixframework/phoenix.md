# phoenixframework/phoenix

> Elixir's full-stack web framework: MVC on the BEAM, with server-rendered real-time UI (LiveView) as its defining bet.

[GitHub repo](https://github.com/phoenixframework/phoenix) ·
[Official website](https://www.phoenixframework.org) ·
[License: MIT](https://github.com/phoenixframework/phoenix/blob/main/LICENSE.md)

## Overview

Phoenix is a web framework for Elixir, created by Chris McCord in 2014 and first released as 1.0 in 2015[^1]. It follows a familiar server-side MVC shape — router, controllers, views, templates — but sits on the Erlang virtual machine (BEAM) and OTP, which gives it lightweight preemptive processes, per-request isolation, and supervision trees for free. The pitch, "peace of mind from prototype to production," leans on the BEAM's decades of soft-real-time telecom heritage rather than on any single feature.

The framework's identity shifted decisively with **LiveView** (2019, integrated into the standard generators by 1.5). LiveView renders interactive UI on the server and ships diffs to the browser over a persistent WebSocket, letting teams build stateful, real-time interfaces with little or no hand-written JavaScript. This is Phoenix's most distinctive and most polarizing trait: it trades the client-heavy SPA model for a stateful server connection per user. Where that trade pays off (dashboards, forms, collaborative UI, internal tools) it removes an entire client/API layer; where it does not (high-latency clients, offline-first, heavy client-side interaction) it can be the wrong tool.

Phoenix is deliberately a thin layer. The database story lives in a separate library (Ecto), the HTTP middleware model is a separate library (Plug), and the web server is pluggable (Bandit or Cowboy). Phoenix wires these together with conventions and code generators rather than absorbing them, which keeps the framework small but means "learning Phoenix" is really learning Elixir, OTP, Ecto, Plug, and LiveView as a stack.

## Getting Started

```bash
mix archive.install hex phx_new      # install the phx.new generator
mix phx.new hello                     # scaffold a project (add --live for LiveView)
cd hello
mix ecto.create                       # create the dev database
mix phx.server                        # http://localhost:4000
```

A minimal LiveView — stateful UI, no client JS:

```elixir
defmodule HelloWeb.CounterLive do
  use HelloWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok, assign(socket, count: 0)}
  end

  def handle_event("inc", _params, socket) do
    {:noreply, update(socket, :count, &(&1 + 1))}
  end

  def render(assigns) do
    ~H"""
    <button phx-click="inc">Count: <%= @count %></button>
    """
  end
end
```

Mount it in the router with `live "/counter", CounterLive`. Each connected client gets its own supervised process holding `socket` state.

## Architecture / How It Works

The request path is built from small, composable pieces:

- **Endpoint** — the top of the tree; a Plug pipeline that terminates the connection, handles the WebSocket upgrade for channels/LiveView, serves static assets, and dispatches to the router.
- **Plug** — the middleware abstraction. Every plug is a function `conn -> conn`. Routers, controllers, and the endpoint are all plug pipelines, so authentication, CSRF, and parsing are just plugs composed in order.
- **Router** — macro-based routing with pipelines (`:browser`, `:api`). Phoenix 1.7 introduced **verified routes** (the `~p` sigil), which check route paths against the router at compile time[^2].
- **Channels / PubSub** — the real-time transport. Channels are bidirectional WebSocket topics; `Phoenix.PubSub` broadcasts across a cluster of BEAM nodes via distributed Erlang, so a message published on one node reaches subscribers on others without external infrastructure.
- **Presence** — CRDT-based tracking of who is connected to a topic, replicated across the cluster without a central store.
- **LiveView** — a stateful process per connection. `mount` runs twice (once for the initial static HTML render, once after the socket connects), then `handle_event`/`handle_info` mutate assigns and Phoenix computes a minimal diff of the HEEx template to push to the browser.

Templates use **HEEx** (HTML-aware EEx), which validates markup at compile time and powers function components (`~H`). Since 1.6 the asset pipeline uses `esbuild` and `tailwind` invoked directly by Mix, with no Node.js toolchain required by default[^3].

Ecto — the database layer — is a separate project (`elixir-ecto/ecto`), not part of Phoenix itself. It provides changesets (validation + casting), a composable query DSL, and migrations. This separation is real: you can run Phoenix without Ecto, and Ecto without Phoenix.

## Production Notes

**LiveView is stateful, and that is an operational fact, not a detail.** Every connected user holds a process (and its assigns) in RAM on a specific node. Consequences: a deploy that drains a node disconnects those sessions (LiveView reconnects and re-`mount`s, so state you didn't persist is lost); large assigns multiplied by many concurrent users is a memory line item; and user-perceived latency is round-trip-bound, so LiveView degrades for clients far from the server or on flaky mobile networks. Keep expensive or reconnect-sensitive state out of the socket, or in the database.

**Clustering is required for cross-node PubSub/Presence.** A single node works out of the box, but multi-node broadcasts, distributed Presence, and horizontal scaling need the BEAM nodes actually connected. In dynamic environments (Kubernetes, Fly.io) this means `libcluster` or equivalent for node discovery; without it, PubSub silently stays node-local and users on different nodes don't see each other's events.

**The BEAM is soft-real-time, not fast at number-crunching.** It excels at massive concurrent I/O with predictable latency and per-process fault isolation. It is comparatively weak at CPU-bound work (image/video processing, ML inference); those belong in NIFs (with care — a crashing NIF takes the whole VM down), ports, or external services.

**Deployment.** The idiomatic path is `mix release` (self-contained OTP release, optionally with the Erlang runtime bundled). Chris McCord joined Fly.io, and Fly.io is a common documented target[^4], but releases run anywhere. Hot code upgrades are technically supported by OTP but rarely used in practice; most teams deploy blue-green or rolling and accept a reconnect.

**Common upgrade friction:**
- 1.3 reorganized generators around **contexts** and renamed the `phoenix` namespace to `phx` — a conceptual shift more than a mechanical one[^5].
- 1.6 dropped the webpack/Node default in favor of esbuild and introduced HEEx; template migration was manual[^3].
- 1.7 added verified routes and moved default styling to Tailwind; older `Routes.*_path` helpers still work but new code uses `~p`[^2].

## When to Use / When Not

**Use when:**
- You want real-time or collaborative UI without building and maintaining a separate SPA + API.
- Your workload is concurrency- and I/O-heavy (many simultaneous connections, chat, presence, dashboards, IoT ingestion).
- You value fault isolation and predictable tail latency, and can invest in learning Elixir/OTP.
- You want one framework to cover HTML apps, JSON APIs, and WebSocket real-time in a single deploy.

**Avoid when:**
- Your team can't absorb the cost of learning Elixir and the actor model, and time-to-first-hire matters more than runtime characteristics.
- The product is offline-first, or clients are high-latency/low-connectivity where a stateful socket hurts.
- The core work is CPU-bound compute (ML, media transcoding) that the BEAM isn't suited to host.
- You need a very large third-party library ecosystem; Hex is healthy but smaller than npm, PyPI, or RubyGems.

## Alternatives

- rails/rails — Ruby full-stack MVC; choose when you want the deepest full-stack ecosystem and hiring pool, and don't need BEAM concurrency. Hotwire covers some LiveView ground.
- django/django — Python batteries-included; choose when proximity to Python's data/ML ecosystem outweighs real-time needs.
- laravel/laravel — PHP full-stack; Livewire directly mirrors LiveView, so choose it in PHP shops wanting the same server-driven UI model.
- hotwired/turbo — not a framework but the server-driven-UI approach (Rails/Hotwire) if you like LiveView's philosophy without the BEAM.
- elixir-ecto/ecto — not an alternative but the database layer Phoenix defers to; worth knowing you can adopt it independently.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015-08 | First stable release; MVC on the BEAM, channels for real-time[^1]. |
| 1.3 | 2017-05 | Contexts, `phx` namespace, umbrella-friendly generators[^5]. |
| 1.4 | 2018-12 | Presence improvements, faster channels, Cowboy 2. |
| 1.5 | 2020-04 | LiveView integrated into generators; Telemetry baked in. |
| 1.6 | 2021-09 | HEEx, esbuild/tailwind, LiveDashboard; webpack/Node dropped[^3]. |
| 1.7 | 2023-02 | Verified routes (`~p`), Tailwind default, function-component layouts[^2]. |

## References

[^1]: Chris McCord, "Phoenix 1.0 released" — 2015-08. https://phoenixframework.org/blog/phoenix-1-0-released
[^2]: Phoenix blog, "Phoenix 1.7 released" — verified routes and Tailwind. https://phoenixframework.org/blog/phoenix-1.7-final-released
[^3]: Phoenix blog, "Phoenix 1.6 released" — HEEx and the esbuild/Node change. https://phoenixframework.org/blog/phoenix-1.6-released
[^4]: Fly.io, "Elixir & Phoenix on Fly.io." https://fly.io/docs/elixir/
[^5]: Chris McCord, "Phoenix 1.3.0 released" — contexts and the `phx` namespace. https://phoenixframework.org/blog/phoenix-1-3-0-released

## Tags

elixir, web-framework, beam, liveview, real-time, mvc, websockets, otp, server-rendered, fullstack
