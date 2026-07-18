# benoitc/hackney

> The veteran Erlang HTTP client — rewritten in 2026 as a process-per-connection
> gen_statem stack with HTTP/2, experimental HTTP/3, WebSocket, and WebTransport.

[GitHub repo](https://github.com/benoitc/hackney) ·
[API docs (hexdocs)](https://hexdocs.pm/hackney) ·
[License: Apache-2.0](https://github.com/benoitc/hackney/blob/master/LICENSE)

## Overview

hackney is a general-purpose HTTP client for Erlang, started by Benoît Chesneau
(also the author of Gunicorn and couchbeam) in July 2012, with 1.0 shipping in
late 2014[^1]. For roughly a decade the 1.x line was the de facto HTTP client of
the BEAM ecosystem: Elixir's HTTPoison is a thin wrapper around it, and it was
long the default Tesla adapter. That ubiquity, not raw quality, is why its ~1.4k
stars understate its actual deployment footprint — it sits in the dependency
tree of a large fraction of Erlang/Elixir applications.

The project's defining event is the **4.0 rewrite (April 2026)**[^2]: a
ground-up redesign that moved every connection into its own `gen_statem`
process, added HTTP/2 (via the `h2` library), opt-in experimental HTTP/3 over
QUIC, WebSocket, and WebTransport clients, TLS verification against Mozilla's
CA bundle by default, and Happy Eyeballs IPv6-first dialing[^3]. The version
number jumped from 1.x straight to 4.x. The tension to understand: hackney 1.x
was old, battle-tested, and carried well-known pool pathologies; hackney 4.x
fixes the architecture but is only months old, requires OTP 27+, and is still
shaking out correctness bugs at a rapid patch cadence (eight releases between
mid-June and mid-July 2026)[^4].

## Getting Started

Rebar3:

```erlang
{deps, [hackney]}.
```

Mix:

```elixir
{:hackney, "~> 4.0"}
```

```erlang
application:ensure_all_started(hackney).

%% Simple GET — body returned directly
{ok, 200, _Headers, Body} = hackney:get(<<"https://httpbin.org/get">>).

%% POST JSON
ReqHeaders = [{<<"content-type">>, <<"application/json">>}],
Payload = <<"{\"key\": \"value\"}">>,
{ok, 200, _, _} = hackney:post(<<"https://httpbin.org/post">>, ReqHeaders, Payload).
```

## Architecture / How It Works

In 4.x each connection is a `gen_statem` process (`hackney_conn`) owning the
socket and protocol state, supervised under `hackney_conn_sup`, with a
connection registry (`hackney_manager`), pool processes, and an Alt-Svc cache
for HTTP/3 discovery[^3]. The connection process monitors its owner: if the
calling process crashes, the connection terminates and the socket is reclaimed —
the OTP-idiomatic answer to the socket leaks that plagued long-running 1.x
deployments.

- **Pooling is TCP-first.** By default only plain TCP connections are pooled;
  an HTTPS request upgrades a pooled TCP connection and closes the TLS session
  after use. `{ssl_pooling, true}` opts into pooling HTTPS/1.1 connections,
  keyed by their exact TLS options. Requests using the default TLS config also
  get TLS 1.3 session-ticket resumption; custom `ssl_options` deliberately never
  do, because a resumed session skips certificate validation against a
  node-wide ticket store[^3].
- **Protocol negotiation** is automatic via ALPN (HTTP/2 when the server
  supports it); HTTP/3 is opt-in per request or globally with
  `{protocols, [http3, http2, http1]}`.
- **Streaming and async** modes exist for both request and response bodies;
  `{async, once}` gives pull-based delivery with HTTP/2 manual flow control.
- **WebTransport** reuses the WebSocket API shape (`wt_*` instead of `ws_*`)
  over HTTP/3.

The HTTP/2 and HTTP/3 machinery lives in separate `h2` and `quic` dependencies
maintained in lockstep by the same author — version bumps there land as hackney
patch releases, so hackney's correctness is coupled to that small satellite
ecosystem[^4].

## Production Notes

**4.x is young — read the changelog before upgrading.** The June–July 2026
patch series fixed, among others: a per-request `hackney_conn` process leak
that could exhaust node memory (#902), HTTP/2 async responses being silently
dropped, connection reuse after an AWS-ALB-style `GOAWAY`, a chunked-decoding
failure on split CRLF reads (#901), and pool shutdown leaking per-host
concurrency slots (#892)[^4]. The cadence signals real maintenance, but teams
on 1.x should treat 4.x as a new client, not a routine bump — there is a
dedicated migration guide[^5].

**OTP 27+ is required** for 4.x. Anything older pins you to the 1.x line, which
still works but carries the legacy architecture.

**1.x pool pathologies are the historical reputation.** Under load, 1.x was
known for `checkout_timeout` errors, pool-manager bottlenecks, and leaked
connections when response bodies were not fully read (`hackney:body/1` or
`with_body` are mandatory hygiene). Much of the Elixir community moved to
Finch/Mint for high-throughput workloads during the long 1.x plateau
(roughly 2021–2025, when releases slowed)[^6]. The 4.x rewrite is explicitly
aimed at that architecture problem; whether it wins those users back is open.

**HTTP/2 flow control needs attention.** Bodies larger than the peer's flow
window block until `WINDOW_UPDATE`, bounded by `send_timeout` (default 30 s)
since 4.7.0; before that they failed with `send_buffer_full`[^4].

**HTTP/3 is experimental** and runs over UDP — corporate networks that drop
UDP will silently fall back or fail; keep `http1`/`http2` in the protocol list.

**One process per connection** costs BEAM process overhead per open connection.
That is normal Erlang design, but a client holding tens of thousands of
concurrent connections should budget for it (and consider gun or raw sockets).

## When to Use / When Not

**Use when:**

- You are on Erlang and want one client covering HTTP/1.1, HTTP/2, WebSocket,
  and (experimentally) HTTP/3 with OTP-supervised connection lifecycles.
- You already depend on hackney via HTTPoison/Tesla and can meet OTP 27+.
- You need streaming or async response delivery as messages, in idiomatic
  Erlang, without writing your own connection process.

**Avoid when:**

- You need years-proven stability today: 4.x is months old and patching fast;
  either pin carefully or stay on 1.x until it settles.
- You are on Elixir and optimizing for throughput with fine pool control —
  Finch (Mint + NimblePool) remains the community default for that profile.
- You are stuck below OTP 27.
- You only make occasional internal HTTP calls — OTP's built-in `httpc` avoids
  the dependency entirely.

## Alternatives

- elixir-mint/mint — process-less, low-level Elixir client; use when you want
  full control of the socket lifecycle inside your own processes.
- sneako/finch — pooled Elixir client built on Mint; use for high-throughput
  Elixir services wanting per-host pool tuning and telemetry.
- ninenines/gun — Erlang HTTP/1.1+HTTP/2+WebSocket client from the Cowboy
  author; use for long-lived, always-up connections rather than request/response.
- puzza007/katipo — libcurl-backed NIF client; use when you need curl's feature
  set or raw performance and accept NIF risk.
- erlang/otp `httpc` — ships with OTP; use for simple, low-volume calls with
  zero extra dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2012-07 | Project started by Benoît Chesneau[^1]. |
| 1.0.0 | 2014-11-30 | First stable release[^1]. |
| 1.18.0 | 2021-09-28 | Late 1.x maintenance line. |
| 1.20.1 | 2023-10-11 | 1.x maintenance during the pre-rewrite plateau. |
| 4.0.0 | 2026-04-16 | Ground-up rewrite: gen_statem connections, HTTP/2, opt-in HTTP/3, WebSocket, WebTransport; OTP 27+[^2]. |
| 4.5.0 | 2026-07-04 | QUERY method (RFC 10008) as a first-class verb[^4]. |
| 4.7.0 | 2026-07-16 | HTTP/2 blocking body sends under flow control; async delivery fixes[^4]. |
| 4.7.2 | 2026-07-17 | quic 1.7.1; clean QUIC connection close[^4]. |

## References

[^1]: benoitc/hackney repository — created 2012-07-02; release 1.0.0 published 2014-11-30. https://github.com/benoitc/hackney
[^2]: hackney 4.0.0 release — 2026-04-16. https://github.com/benoitc/hackney/releases/tag/4.0.0
[^3]: hackney README and Design Guide (process-per-connection, pooling, TLS, HTTP/3). https://github.com/benoitc/hackney/blob/master/guides/design.md
[^4]: hackney NEWS.md changelog (4.4.x–4.7.x, June–July 2026). https://github.com/benoitc/hackney/blob/master/NEWS.md
[^5]: hackney migration guide (1.x → 4.x). https://github.com/benoitc/hackney/blob/master/guides/MIGRATION.md
[^6]: Finch — Elixir HTTP client focused on performance, built on Mint. https://github.com/sneako/finch

## Tags

erlang, elixir, http-client, http2, http3, quic, websocket, connection-pooling, networking, beam
