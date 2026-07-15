# phoenixframework/phoenix_live_view

> Rich, real-time UIs rendered as server HTML and diffed over a WebSocket — no hand-written client state.

[GitHub repo](https://github.com/phoenixframework/phoenix_live_view) ·
[Hex package](https://hex.pm/packages/phoenix_live_view) ·
[License: MIT](https://github.com/phoenixframework/phoenix_live_view/blob/main/LICENSE.md)

## Overview

Phoenix LiveView is an Elixir library that builds interactive web UIs without a
separate frontend application. The server holds the authoritative UI state in a
long-lived process, renders HTML, and pushes minimal diffs to the browser over a
persistent WebSocket. User events (`phx-click`, `phx-submit`, …) travel back to
the same process, which recomputes state and re-renders. It was created by Chris
McCord and first announced in December 2018[^1]; it now ships by default in new
Phoenix applications.

The defining tradeoff is **server-centric state over a stateful connection**. You
give up the split-brain complexity of keeping a client store in sync with a
server API — in exchange, every interaction is a network round trip, and each
connected user occupies a live process (with its assigns) in server memory. On
the BEAM (Erlang VM) this is unusually cheap: processes are lightweight and a
single node routinely holds tens of thousands of concurrent connections. But the
model is fundamentally latency- and connection-bound, which is a poor fit for
offline-first apps or highly interactive client-side interactions.

LiveView is best understood as the Elixir answer to the same problem
Hotwire (Rails) and Livewire (Laravel) address: server-rendered interactivity.
Its differentiator is change tracking made possible by Elixir's immutability and
compile-time template analysis, so the wire payload after the first render is
just the values that actually changed.

## Getting Started

LiveView ships with new Phoenix apps. After installing Elixir:

```bash
mix archive.install hex phx_new
mix phx.new demo
cd demo && mix phx.server
```

A minimal LiveView — a server-held counter:

```elixir
defmodule DemoWeb.CounterLive do
  use DemoWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok, assign(socket, count: 0)}
  end

  def handle_event("inc", _params, socket) do
    {:noreply, update(socket, :count, &(&1 + 1))}
  end

  def render(assigns) do
    ~H"""
    <button phx-click="inc">Count: {@count}</button>
    """
  end
end
```

Route it with `live "/counter", CounterLive` in your router. `mount/3` runs
twice — once for the static HTTP render, once when the WebSocket connects.

## Architecture / How It Works

Each LiveView is an OTP process (a `GenServer` under the hood) holding the
socket's `assigns`. The lifecycle:

1. **Dead render (HTTP).** The first request renders the LiveView as ordinary
   server HTML — good for first paint, SEO, and no-JS fallback. `mount/3` is
   called with `connected?(socket)` false.
2. **Connected mount (WebSocket).** The client's `LiveSocket` opens a Phoenix
   Channel; `mount/3` runs again, now stateful. State that must survive the
   handshake is passed through the session or reloaded here.
3. **Event loop.** DOM events matching `phx-*` bindings are sent to the process,
   handled by `handle_event/3`; server-side messages arrive via `handle_info/2`
   (e.g. PubSub broadcasts). Each returns updated assigns.
4. **Diff + patch.** `~H` (HEEx) templates are compiled into static and dynamic
   parts. LiveView tracks which assigns changed and sends only the affected
   dynamic segments; the client patches the DOM with `morphdom`.

**HEEx** is the templating layer: HTML-aware, with function components, slots,
compile-time validation, and verified routes. Function components are stateless
(`attr`/`slot` declared); **LiveComponents** are stateful, addressable units
with their own `handle_event/3`, useful for isolating parts of a page.

**Streams** (introduced during the 0.18 line) exist specifically to render large
or unbounded collections without holding every item in server memory — the
canonical fix for the assigns-memory problem below.

`Phoenix.LiveView.JS` provides client-side commands (show/hide, toggle class,
push events) for optimistic updates and transitions without writing custom
JavaScript. `phx-hook` is the escape hatch when you genuinely need JS.

## Production Notes

- **Memory scales with connections.** Every connected client is a process whose
  assigns live in RAM for the connection's lifetime. Large assigns (full lists,
  big structs) multiply across users. Mitigations: `stream/3` for collections,
  `temporary_assigns` to drop data after render, and keeping only IDs/derived
  state in assigns.
- **Latency is the UX ceiling.** Interactions round-trip to the server, so
  perceived responsiveness tracks network RTT. For high-latency or flaky
  connections, lean on `Phoenix.LiveView.JS` and CSS for optimistic feedback;
  LiveView will not feel like a local-first SPA.
- **Persistent connections constrain deployment.** WebSockets rule out most
  request/response serverless platforms. You want a always-on BEAM node (Fly.io,
  Gigalixir, bare VMs, Render, K8s). Load balancers need WebSocket support and
  sane idle timeouts.
- **Reconnects re-run mount.** A dropped socket re-mounts and rebuilds state, so
  `mount/3` must be able to reconstruct everything — don't assume a single
  continuous session. Form recovery (`phx-auto-recover`) restores in-flight input
  after reconnect.
- **Node affinity, not sticky HTTP.** The stateful process lives on one node;
  cross-node coordination uses `Phoenix.PubSub` (distributed Erlang or a PubSub
  adapter), not a shared cache. There is no built-in horizontal session sharing —
  reconnection to another node re-mounts fresh.
- **Testing is browserless.** `Phoenix.LiveViewTest` drives mount/render/events
  in-process, which is fast and reliable; full end-to-end still needs a real
  browser (the repo itself uses Playwright for its JS e2e suite).
- **Upgrade friction historically.** LiveView spent years in the 0.x range with
  real breaking changes between minors (HEEx replacing Leex around 0.16, the move
  to declarative assigns and function components in the 0.18 line). The 1.0
  release stabilized the public API; pre-1.0 tutorials can be actively misleading.

## When to Use / When Not

**Use when:**
- Your backend is already Elixir/Phoenix and you want interactivity without a
  separate SPA and API layer.
- The app is CRUD-, form-, dashboard-, or real-time-feed-shaped (live updates,
  presence, notifications) where server-pushed state is natural.
- You want server-rendered first paint and progressive enhancement by default.

**Avoid when:**
- You need offline support or must keep working through long connection drops.
- The UI is intensely interactive client-side (drawing, drag-heavy editors,
  games) where per-interaction round trips are unacceptable.
- Your team/stack isn't Elixir and adopting the BEAM only for the frontend is not
  justified.
- You must deploy on request/response serverless with no persistent sockets.

## Alternatives

- hotwired/turbo — the Rails/Hotwire take on server-HTML-over-the-wire; use when
  your backend is Ruby.
- livewire/livewire — Laravel/PHP equivalent of the stateful server-component
  model; use in a PHP stack.
- bigskysoftware/htmx — hypermedia-driven interactivity with no persistent
  connection and no server-held state; use when you want the philosophy without
  WebSockets or the BEAM.
- dotnet/aspnetcore (Blazor Server) — .NET's stateful server-rendered UI over a
  SignalR connection; the closest architectural analog outside Elixir.
- facebook/react + an API — use when you need a rich offline-capable client and
  can accept maintaining client/server state separately.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-12 | Initial announcement by Chris McCord[^1]. |
| 0.1.0 | 2019 | First public Hex releases; `~L` (Leex) templates. |
| 0.16 | 2021 | HEEx (`~H`) introduced, replacing Leex[^2]. |
| 0.18 | 2022 | Declarative assigns, function components, slots. |
| 0.18.x | 2023 | LiveView streams for large collections. |
| 0.20 | 2023 | Continued 0.x hardening ahead of 1.0. |
| 1.0.0 | 2024 | Stable public API[^3]. |

## References

[^1]: "Phoenix LiveView: Interactive, Real-Time Apps. No Need to Write JavaScript." DockYard, 2018-12-12. https://dockyard.com/blog/2018/12/12/phoenix-liveview-interactive-real-time-apps-no-need-to-write-javascript
[^2]: Phoenix LiveView HexDocs (HEEx templating, components, streams). https://phoenix-live-view.hexdocs.pm
[^3]: Phoenix LiveView CHANGELOG. https://github.com/phoenixframework/phoenix_live_view/blob/main/CHANGELOG.md

## Tags

elixir, phoenix, liveview, server-rendered, websocket, real-time, web-framework, beam, hotwire-alternative, ui, functional
