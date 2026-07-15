# elixir-tesla/tesla

> A middleware-based HTTP client for Elixir with pluggable adapters — Faraday's idea ported to the BEAM.

[GitHub repo](https://github.com/elixir-tesla/tesla) ·
[Docs (HexDocs)](https://hexdocs.pm/tesla/) ·
[Hex package](https://hex.pm/packages/tesla) ·
[License: MIT](https://github.com/elixir-tesla/tesla/blob/master/LICENSE)

## Overview

Tesla is an HTTP client library for Elixir built around two extension points: a
composable **middleware** stack (that transforms requests and responses) and a
swappable **adapter** (that performs the actual network I/O). It was inspired by
Ruby's Faraday[^1], and it fills the same niche in the Elixir ecosystem — the
library most people reach for when building a typed, reusable API client rather
than making one-off requests. The project dates to 2015 and reached its stable
`1.x` line in 2019.

The defining design choice is that Tesla is not itself an HTTP stack. It defines
a request/response pipeline and delegates the socket work to an adapter such as
Mint, Finch, Hackney, Gun, or ibrowse. That decoupling is the whole value
proposition: you write middleware once (auth, JSON, retries, logging) and can
change the underlying transport without touching client code. The cost is that
Tesla's behavior — connection pooling, HTTP/2 support, TLS validation, timeout
semantics — is only as good as the adapter you pick, and the defaults are a
trap (see Production Notes).

Tesla is widely used as a dependency: many published Elixir API SDKs and the
OpenAPI generator's Elixir target emit Tesla-based clients. It is
maintained but mature — releases are infrequent and mostly additive, which for a
client library is closer to a feature than a warning sign.

## Getting Started

Add the dependency (and an adapter — do not ship the default) in `mix.exs`:

```elixir
defp deps do
  [
    {:tesla, "~> 1.11"},
    {:jason, "~> 1.4"},   # for the JSON middleware
    {:mint, "~> 1.0"}     # a real adapter
  ]
end
```

Set a non-`httpc` adapter globally in `config/config.exs`:

```elixir
config :tesla, adapter: Tesla.Adapter.Mint
```

Build a client by composing middleware, then reuse it across requests:

```elixir
client =
  Tesla.client([
    {Tesla.Middleware.BaseUrl, "https://api.example.com"},
    {Tesla.Middleware.Headers, [{"authorization", "Bearer " <> token}]},
    Tesla.Middleware.JSON,
    {Tesla.Middleware.Retry, max_retries: 3}
  ])

{:ok, %Tesla.Env{status: 200, body: body}} = Tesla.get(client, "/users/1")
```

The idiomatic pattern is to wrap this in a module with functions like
`get_user/1`, exposing a typed API and keeping the middleware list private.

## Architecture / How It Works

A request flows through the middleware list in order, hits the adapter, then
unwinds back through the same middleware in reverse. Each middleware is a module
implementing `call(env, next, opts)` — it may mutate the `%Tesla.Env{}`, call
`Tesla.run(env, next)` to continue down the stack, then inspect the response on
the way back. This is the classic onion/plug model; middleware like `Retry` and
`Timeout` work by wrapping the `next` call rather than by transforming a single
value.

There are two ways to assemble a client. `Tesla.client/1,2` builds a **runtime**
client (a plain struct holding the middleware list and adapter) — this is the
recommended approach and what most current code uses. The older approach is
`use Tesla` with `plug`/`adapter` macros, which builds the stack at **compile
time** into the module. The macro form still works but couples the middleware
list to the module definition and makes per-instance configuration awkward;
newer guidance favors the runtime `client/1` form.

Adapters implement a single `call(env, opts)` returning `{:ok, env}` or
`{:error, reason}`. Because the adapter is just a module reference, it can be
overridden per-client (`Tesla.client(middleware, adapter)`), which is how the
test adapter (`Tesla.Mock`) substitutes canned responses without touching the
network. Streaming request/response bodies is supported but is adapter-specific —
not every adapter honors a streaming body.

## Production Notes

**The default adapter is `:httpc` and you must not ship it.** The README says so
explicitly[^2]: Erlang's built-in `httpc` does not validate SSL certificates by
default, among other issues, and Tesla will not fix this for backward-compat
reasons. A fresh project that forgets to configure an adapter silently makes
insecure TLS requests. Always set Mint, Finch, or Hackney before production.

**Adapter choice is a real architectural decision, not a detail.** Hackney
(`:hackney`) is the historical default many libraries pin, but its pooling has
been a recurring source of connection-leak and checkout-timeout reports. Finch
(built on Mint + NimblePool) is the current recommendation for
throughput-sensitive services and gives you explicit, named connection pools —
but Finch pools must be started in your supervision tree and referenced by name,
which is extra wiring Tesla does not do for you. Mint is a good default for
moderate load. Switching adapters can change timeout, redirect, and pooling
behavior even with identical middleware.

**Middleware order matters and is easy to get wrong.** Because the stack is
symmetric, placement determines what each middleware sees. `Tesla.Middleware.JSON`
must sit above adapter-level concerns; `Retry` placed above `Logger` will not
log the intermediate failed attempts, and placed below it will log each retry.
Auth headers added after a caching or logging middleware may leak into logs.

**Timeouts are layered and independent.** `Tesla.Middleware.Timeout` enforces a
deadline in a supervised task regardless of the adapter, while the adapter also
has its own connect/receive timeouts. These do not coordinate; a request can be
governed by whichever fires first, and misconfiguring one while trusting the
other is a common cause of "hung" requests.

**Compile-time clients recompile on config change.** If you use the `use Tesla`
macro form with module attributes for config, changing that config requires
recompilation. Runtime clients avoid this. The v0→v1 migration guide[^3] covers
the shift in idioms if you are upgrading an old codebase.

## When to Use / When Not

**Use when:**
- You are building a reusable API client/SDK and want typed wrapper functions.
- You need cross-cutting behavior (auth, retries, JSON, logging) applied uniformly.
- You want to keep the option of swapping the HTTP transport later.
- You want a first-class test story via `Tesla.Mock`.

**Avoid when:**
- You just need a couple of one-off requests — Req offers a simpler, batteries-included API.
- You want opinionated, secure defaults out of the box — Tesla's default adapter is deliberately insecure.
- You need guaranteed HTTP/2, streaming, or pooling behavior — that lives in the adapter, so evaluate the adapter, not Tesla.

## Alternatives

- wojtekmach/req — higher-level Elixir client with safe defaults, built on Finch; preferred for most new application code over hand-assembling Tesla middleware.
- elixir-mint/mint — the low-level functional HTTP/1+HTTP/2 client; use directly when you want no middleware layer and full control.
- sneako/finch — Mint + connection pooling; use when you need explicit high-throughput pools (also usable as a Tesla adapter).
- benoitc/hackney — Erlang HTTP client; use when a dependency already requires it, but watch pool behavior.
- lostisland/faraday — the Ruby library Tesla is modeled on; relevant only if you are porting concepts between ecosystems.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015–2018 | Original macro-based API (`use Tesla`), compile-time stacks. |
| 1.0 | 2019-01 | Stable API; `%Tesla.Env{}`, runtime `Tesla.client/1`, adapter behaviour. |
| 1.4 | 2021 | Adapter and middleware refinements; broader adapter support. |
| 1.7–1.11 | 2023–2025 | Ongoing additive releases; OpenAPI-style query serialization, guides. |

_Exact minor-version dates vary; consult the Hex changelog for specifics._[^4]

## References

[^1]: Tesla README — "Inspired by Faraday from Ruby." https://github.com/elixir-tesla/tesla#readme
[^2]: Tesla README — ":httpc as default Adapter" warning on missing SSL validation. https://github.com/elixir-tesla/tesla#getting-started
[^3]: Tesla guide — "Migrating from v0 to v1.x." https://hexdocs.pm/tesla/
[^4]: Tesla on Hex — release history and changelog. https://hex.pm/packages/tesla

## Tags

elixir, erlang, beam, http-client, middleware, api-client, networking, adapters, otp, faraday-inspired
