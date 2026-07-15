# elixir-plug/plug

> The connection specification that every Elixir web application — Phoenix included — is built on top of.

[GitHub repo](https://github.com/elixir-plug/plug) ·
[Hex package](https://hex.pm/packages/plug) ·
[License: Apache-2.0](https://github.com/elixir-plug/plug/blob/main/LICENSE)

## Overview

Plug is two things at once: a specification for composing web applications out of plain functions, and a small set of connection adapters that bind that specification to actual web servers running on the Erlang VM (BEAM). It is the Elixir analogue of Ruby's Rack or Python's WSGI/ASGI — the thin, standardized seam between "your request-handling code" and "the thing that speaks HTTP over a socket."[^1]

The central abstraction is a *plug*: a function that receives a `%Plug.Conn{}` struct and returns a (transformed) `%Plug.Conn{}`. Because both the input and output are the same connection type, plugs compose by ordinary function chaining, and a pipeline of plugs is itself a plug. This is the whole design. Everything else — routing, sessions, CSRF protection, static file serving, parsers — is a plug that fits into the same interface.

Plug's defining characteristic is that almost nobody uses it directly, yet nearly every Elixir web stack depends on it. Phoenix is a Plug pipeline underneath its routing, channels, and LiveView layers.[^2] That gives Plug an unusual position: it is foundational infrastructure with a small, stable API surface and low churn, while the visible churn in the ecosystem happens one layer up (Phoenix) or one layer down (the web server adapters). At roughly 3k stars and 600 forks the raw popularity numbers understate its reach — the dependency graph, not the star count, is where its importance lives.

## Getting Started

Plug needs a web server adapter. Two are commonly used: Cowboy (Erlang-based, long the default) and Bandit (pure Elixir, newer).[^3]

```elixir
# mix.exs
def deps do
  [{:plug_cowboy, "~> 2.0"}]   # or {:bandit, "~> 1.0"}
end
```

```elixir
defmodule MyPlug do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    conn
    |> put_resp_content_type("text/plain")
    |> send_resp(200, "Hello world")
  end
end

# Start under a supervision tree in production; this is the throwaway form:
webserver = {Plug.Cowboy, plug: MyPlug, scheme: :http, options: [port: 4000]}
{:ok, _} = Supervisor.start_link([webserver], strategy: :one_for_one)
```

`MyPlug` is a *module plug*: it defines `init/1` (called once, at compile/boot time, to prepare options) and `call/2` (called per request). A *function plug* is just a `fun(conn, opts)` with the same contract.

## Architecture / How It Works

**`Plug.Conn`.** The connection is an immutable struct that holds request fields (`host`, `method`, `path_info`, `req_headers`, params), response fields (`status`, `resp_body`, `resp_headers`), and adapter state. Every helper in `Plug.Conn` (`put_resp_header/3`, `send_resp/3`, `fetch_query_params/1`, …) returns a new copy. There is no mutable request object; state is threaded explicitly through the pipeline. The struct is a *direct interface to the underlying server* — `send_resp/3` flushes the status and body to the client immediately, which is what makes response streaming natural but also means you cannot un-send a response once flushed.

**Pipelines and `halt/1`.** A pipeline is an ordered list of plugs. Each is invoked in turn on the connection. Calling `halt/1` sets a flag that stops downstream plugs from running — this is how authentication, CSRF, and rate-limiting plugs short-circuit a request. `Plug.Builder` is the macro that compiles a `plug ...` list into a single `call/2`.

**`Plug.Router`.** Routing is done by `use Plug.Router` plus `plug :match` and `plug :dispatch`. The interesting part is compile-time: all routes are compiled into one function and the BEAM's pattern-match compiler turns them into a decision tree rather than a linear scan, so lookup cost does not grow linearly with route count.[^1] The tradeoff is that an unmatched request raises a `FunctionClauseError` unless you define a catch-all `match _`.

**Adapters.** `Plug.Cowboy` and `Bandit` implement the adapter behaviour that connects `Plug.Conn` to a real socket. Swapping servers is, in principle, a one-line change in the child spec. Concurrency is handled by the BEAM: one lightweight process per request, so plugs are written as ordinary synchronous code with no async coloring.

**Batteries.** Plug ships a standard library of plugs: `Plug.Parsers` (body parsing by content-type, including multipart uploads), `Plug.Session`, `Plug.CSRFProtection`, `Plug.Static`, `Plug.SSL`, `Plug.RequestId`, `Plug.Logger`, `Plug.Telemetry`, `Plug.BasicAuth`, `Plug.Head`, `Plug.MethodOverride`, and `Plug.RewriteOn`. Since v1.14 a connection `upgrade` API provides WebSocket support (typically paired with `websock_adapter`).[^4]

## Production Notes

**Pipeline order is load-bearing.** Because plugs run top to bottom, ordering is a correctness concern, not a style one. `Plug.Static` must come before your router so asset requests short-circuit before routing; `Plug.CSRFProtection` requires `Plug.Session` (and a fetched session) ahead of it; parsers must run before any plug that reads params.

**The request body is read once.** `Plug.Parsers` (or a manual `read_body/2`) consumes the body stream. Read it twice and the second read is empty. Multipart/file-upload handling, size limits, and which content-types you parse all live in the `Plug.Parsers` options — misconfiguring `:length` or omitting a parser is a common source of "why is my JSON body empty" bugs.

**Adapter migration is the real churn, not Plug.** Plug's own API is stable and rarely breaks. What moves is the server layer: the ecosystem has been shifting from Cowboy to Bandit, and Phoenix now defaults to Bandit for new projects.[^3] Bandit is generally a drop-in via the child spec, but adapter-specific behavior (HTTP/2 details, `:protocol_options`, timeout knobs, WebSocket handling) does not transfer verbatim between Cowboy and Bandit. Pin and test the adapter explicitly.

**`init/1` runs at compile time in the module-plug/Phoenix path.** Options returned from `init/1` are frozen into the compiled pipeline, so you cannot read runtime configuration (env vars, application config resolved at boot) inside `init/1` and expect it to change without a recompile. Runtime config must be read inside `call/2`.

**Observability.** Use `Plug.Telemetry` and `Plug.RequestId` rather than hand-rolling. Telemetry events are the supported integration point for metrics/tracing; `Plug.Logger` alone is not enough for production tracing.

**Support policy.** The maintainers ship bug fixes only for the current minor and security patches for roughly the last four minors, so staying within a few minors of latest is expected if you want fixes.[^5]

## When to Use / When Not

**Use when:**
- You are building anything HTTP on the BEAM and want the standard connection contract every library expects.
- You want a minimal router/pipeline without adopting the full Phoenix stack (small services, webhooks, health endpoints, API gateways).
- You are writing reusable web middleware for the Elixir ecosystem — a plug is the portable unit.

**Avoid / reconsider when:**
- You want routing, code generation, channels, LiveView, and an opinionated project layout — reach for phoenixframework/phoenix, which gives you Plug plus all of that.
- You need raw control over the HTTP server (custom protocol handling, non-request workloads) — talk to Cowboy or Bandit directly and skip the Plug abstraction.
- You are not on the Erlang VM — Plug is BEAM-only; the concept transfers but the code does not.

## Alternatives

- phoenixframework/phoenix — use instead when you want a full framework; it is Plug plus routing, LiveView, channels, and generators, not a competitor.
- mtrudel/bandit — the pure-Elixir server adapter; choose it over `plug_cowboy` for new projects wanting an all-Elixir stack.
- elixir-plug/plug_cowboy — the Cowboy adapter; use when you need Cowboy's maturity or existing Cowboy tuning.
- ninenines/cowboy — use directly when you want to bypass Plug and handle raw Erlang HTTP yourself.
- rack/rack — the Ruby analogue of the same idea; relevant only as a conceptual reference, not an Elixir option.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2013-11-15 | Initial development at Plataformatec (José Valim et al.).[^6] |
| 1.0 | 2015 | Stable connection spec and `Plug.Conn` API frozen.[^1] |
| 1.14 | 2022 | Connection `upgrade` API adds WebSocket support out of the box.[^4] |
| 1.x (Bandit era) | 2023–2024 | Bandit matures to 1.0 and becomes the recommended Elixir-native adapter alongside Cowboy.[^3] |
| 1.20 | 2026 | Current release line; bug fixes on latest, security patches on prior four minors.[^5] |

## References

[^1]: Plug README and hexdocs — "A specification for composing web applications with functions." https://hexdocs.pm/plug/
[^2]: Phoenix framework — built on Plug for request/response handling. https://hexdocs.pm/phoenix/plug.html
[^3]: Bandit, a pure-Elixir HTTP server implementing the Plug adapter interface. https://hexdocs.pm/bandit/ and https://github.com/mtrudel/bandit
[^4]: Plug README — "Plug v1.14 includes a connection `upgrade` API … WebSocket support out of the box." https://github.com/elixir-plug/plug#hello-world-websockets
[^5]: Plug README, "Supported Versions" — bug fixes for current release, security patches for the last four minor releases. https://github.com/elixir-plug/plug#supported-versions
[^6]: GitHub repository metadata, `created_at` 2013-11-15. https://github.com/elixir-plug/plug

## Tags

elixir, erlang-vm, beam, web, http, middleware, connection-adapter, request-pipeline, phoenix, plug-spec, router
