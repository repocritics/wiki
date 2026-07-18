# ninenines/cowboy

> Small, fast HTTP server for Erlang/OTP — one process per request, and the server that carried the BEAM web ecosystem for a decade.

[GitHub repo](https://github.com/ninenines/cowboy) ·
[Official website](https://ninenines.eu) ·
[License: ISC](https://github.com/ninenines/cowboy/blob/master/LICENSE)

## Overview

Cowboy is an HTTP server for Erlang/OTP, started by Loïc Hoguin in March 2011 and maintained by his one-person consultancy Nine Nines[^1]. It aims to be a *complete* HTTP stack — HTTP/1.1, HTTP/2, Websocket, REST semantics, and experimental HTTP/3 — in a deliberately small code base optimized for low latency and low memory, largely by working exclusively with Erlang binary strings rather than char lists[^2]. It is not a web framework: there are no templates, no ORM, no sessions. It gives you routing, protocol handling, and handler behaviours, and expects you to bring the rest.

Its reach is much larger than its star count suggests. Via the `plug_cowboy` adapter it was the default server under Elixir's Plug and Phoenix for roughly a decade, and it serves the HTTP APIs of infrastructure like RabbitMQ's management plugin and VerneMQ. Newer Phoenix generators default to Bandit, a pure-Elixir alternative, but `plug_cowboy` remains supported and widely deployed.

The defining tension is the project's shape: essentially one maintainer, funded by sponsorship and commercial support, with a release cadence that swings between multi-year gaps (2.9 in 2021-05 → 2.10 in 2023-04) and bursts (three releases across 2026 so far)[^3]. The code is mature, conservative, and well tested — Cowboy's test suites double as a de facto HTTP conformance harness — but if you need a feature, you may wait, sponsor, or fork.

## Getting Started

```erlang
%% rebar.config
{deps, [cowboy]}.
```

A minimal handler and listener (Cowboy 2.x API):

```erlang
-module(hello_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req0, State) ->
    Req = cowboy_req:reply(200,
        #{<<"content-type">> => <<"text/plain">>},
        <<"Hello, world!">>,
        Req0),
    {ok, Req, State}.
```

```erlang
%% in your application start/2
Dispatch = cowboy_router:compile([
    {'_', [{"/", hello_handler, []}]}
]),
{ok, _} = cowboy:start_clear(http_listener,
    [{port, 8080}],
    #{env => #{dispatch => Dispatch}}).
```

`start_tls/3` is the TLS equivalent; ALPN selects HTTP/2 automatically. From Elixir, you normally consume Cowboy through `plug_cowboy` rather than this API directly.

## Architecture / How It Works

Cowboy is the top of a three-repo stack, all under the ninenines org:

- **Ranch**[^4] — socket acceptor pool. A listener owns the socket, a configurable set of acceptors accept connections, and each connection gets its own Erlang process. Because Ranch is a separate library, Cowboy embeds cleanly into any OTP application — it is a supervised set of processes, not a standalone daemon.
- **cowlib**[^5] — pure protocol-parsing library (HTTP/1.1, HPACK, Websocket framing, etc.), built on binary pattern matching.
- **Cowboy** — connection processes run a protocol module (`cowboy_http`, `cowboy_http2`, `cowboy_http3`), which feeds requests through a **stream handler** chain (`cowboy_stream` behaviour). The default `cowboy_stream_h` spawns a *separate process per request*, so a crashing handler kills only that request, never the connection. Optional stream handlers bolt on response compression (`cowboy_compress_h`), metrics (`cowboy_metrics_h`), and live request tracing (`cowboy_tracer_h`).

Above the streams sits a small middleware chain — by default just `cowboy_router` (host + path matching with bindings and constraints, compiled dispatch tables) and `cowboy_handler`. Handlers implement one of a few behaviours: plain `cowboy_handler` (`init/2`), `cowboy_rest` (a Webmachine-style HTTP decision flowchart driven by callbacks like `content_types_provided` and `resource_exists`), `cowboy_websocket`, or `cowboy_loop` for long-polling/SSE-style push.

The request object (`Req`) is a map in 2.x, and bodies are never read implicitly: `cowboy_req:read_body/1,2` streams chunks on demand, and `stream_reply`/`stream_body` do the same for responses. HTTP/3 arrived experimentally in 2.11 (2024-01) on top of the `quicer` NIF (msquic) and is opt-in at build time; WebTransport support is layered on the same stack.

## Production Notes

**One process per request is a tradeoff.** It buys isolation and clean OTP supervision semantics, but adds a process spawn plus message passing per request. Benchmarks of Bandit vs `plug_cowboy` in the Elixir world typically show Bandit ahead on HTTP/1.1 throughput. Cowboy's latency remains fine for almost all workloads; if you are chasing hello-world req/s numbers, this is not the server that wins them.

**HTTP/2 defaults are conservative.** Large uploads over HTTP/2 can be throttled by flow-control window defaults; tune `initial_stream_window_size`, `initial_connection_window_size`, and `max_frame_size_received` in the protocol options if body-heavy HTTP/2 traffic matters to you.

**Websocket at scale.** Each connection is a process. For large fleets of mostly-idle connections, set `hibernate` in the websocket options to shrink per-process memory, and think twice before enabling `compress` (permessage-deflate) — the zlib context costs real memory per connection.

**Read your bodies, mind your timeouts.** `read_body` has default length/period limits and returns `{more, ...}` — code that assumes one call reads everything breaks on large bodies. `idle_timeout`, `request_timeout`, and `inactivity_timeout` are all configurable and worth reviewing behind load balancers; PROXY protocol support comes via Ranch's `proxy_header`.

**The 1.x → 2.0 migration was a hard break.** 2.0 (2017) replaced the `#http_req{}` record with an opaque map, removed `onrequest`/`onresponse` hooks in favor of stream handlers, and changed handler signatures. Consequence today: a large fraction of older Cowboy tutorials and StackOverflow answers describe the dead 1.x API. Trust only the ninenines.eu docs for your exact version[^2].

**Build-system friction.** Cowboy itself builds with erlang.mk, Hoguin's own tool. As a Hex dependency under rebar3 or Mix this is invisible, but contributing or building docs/examples means learning erlang.mk. HTTP/3 additionally requires the `quicer` NIF toolchain.

**Sustainability.** Active maintenance is real — 2.15, 2.16, and 2.17 all shipped in 2026[^3] — but the bus factor is effectively one. Budget for that in risk assessments the same way you would for any single-maintainer critical dependency.

## When to Use / When Not

**Use when:**
- You are on the BEAM (Erlang or Elixir) and need a production HTTP/Websocket server embedded in an OTP application.
- You need long-lived connections at scale — Websocket, SSE, long-polling — where per-connection processes and supervision are the right model.
- You want REST semantics done correctly via `cowboy_rest`'s decision flowchart instead of hand-rolled content negotiation.
- You need HTTP/2 today and a path to HTTP/3 without leaving Erlang.

**Avoid when:**
- You are not on the BEAM — Cowboy is an Erlang library, not a general-purpose server like nginx or Caddy.
- You want a batteries-included web framework; Cowboy is the layer Phoenix sits on, not a Phoenix substitute.
- You are starting a fresh Elixir/Phoenix app and have no Cowboy-specific needs — Bandit is now the generator default and integrates with Plug telemetry more natively.
- Peak HTTP/1.1 throughput on simple responses is your primary metric.

## Alternatives

- mtrudel/bandit — pure-Elixir Plug server; use instead for new Phoenix/Plug apps wanting an all-Elixir stack and better HTTP/1.1 throughput.
- elli-lib/elli — minimalist Erlang HTTP/1.1 server; use when you want the smallest possible overhead and need neither HTTP/2 nor Websocket.
- erlyaws/yaws — veteran full-featured Erlang web server; use when you want a standalone server with dynamic-content facilities rather than an embeddable library.
- mochi/mochiweb — the pre-Cowboy generation of lightweight Erlang HTTP; use only to maintain existing codebases.
- erlang/otp — the built-in `inets` httpd; use when zero external dependencies outweigh ergonomics and features.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2011-03 | Project started by Loïc Hoguin[^1]. |
| 1.0 | 2014-08 | First stable release: HTTP/1.1, Websocket, REST handlers[^3]. |
| 2.0 | 2017-10 | Ground-up rewrite: HTTP/2, map-based `Req`, one process per request, stream handlers. Breaking migration from 1.x[^6]. |
| 2.10 | 2023-04 | Maintenance release after a two-year gap (2.9 was 2021-05)[^3]. |
| 2.11 | 2024-01 | Experimental HTTP/3 over QUIC via the quicer NIF[^7]. |
| 2.17 | 2026-07 | Current release; third release of 2026 (after 2.15 in May, 2.16 in June)[^3]. |

## References

[^1]: GitHub repository metadata — created 2011-03-09; Nine Nines. https://github.com/ninenines/cowboy
[^2]: Cowboy 2.17 user guide and function reference. https://ninenines.eu/docs/en/cowboy/2.17/guide/
[^3]: Release tag dates from the GitHub API (1.0.0: 2014-08-01, 2.0.0: 2017-10-03, 2.9.0: 2021-05-12, 2.10.0: 2023-04-28, 2.15.0: 2026-05-12, 2.16.0: 2026-06-08, 2.17.0: 2026-07-02). https://github.com/ninenines/cowboy/tags
[^4]: Ranch — socket acceptor pool for TCP protocols. https://github.com/ninenines/ranch
[^5]: cowlib — support library for manipulating web protocols. https://github.com/ninenines/cowlib
[^6]: Cowboy 2.0.0 release. https://github.com/ninenines/cowboy/releases/tag/2.0.0
[^7]: Cowboy 2.11.0 release. https://github.com/ninenines/cowboy/releases/tag/2.11.0

## Tags

erlang, otp, beam, http-server, websocket, http2, http3, rest, networking, low-latency
