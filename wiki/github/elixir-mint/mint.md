# elixir-mint/mint

> A process-less HTTP/1.1 and HTTP/2 client for Elixir — the connection is an
> immutable data structure you own, not a process you talk to.

[GitHub repo](https://github.com/elixir-mint/mint) ·
[Documentation](https://hexdocs.pm/mint) ·
License: Apache-2.0

## Overview

Mint is a low-level HTTP client for Elixir by Eric Meadows-Jönsson (creator of
Hex) and Andrea Leopardi (Elixir core team), announced on the official Elixir
blog in February 2019[^1]. Its design premise is unusual for the BEAM: where
every notable Erlang/Elixir HTTP client before it (hackney, gun, httpc) spawns
processes to own connections, Mint gives you a functional, immutable `conn`
struct wrapping a raw TCP or SSL socket — the README's own framing is
accurate: `:gen_tcp`/`:ssl` with an understanding of HTTP/1.1 and HTTP/2.

The consequence is that Mint does almost nothing for you: no pooling, no
retries, no redirects, no synchronous request API, no WebSockets. You decide
which process owns which socket and how responses are assembled from streamed
parts. That makes Mint the wrong first choice for application code and the
right substrate for libraries: Finch (and therefore Req, the current default
Elixir HTTP client) is built on Mint plus NimblePool[^2], and mint_web_socket
extends it to WebSockets[^3].

At ~1.4k stars it is not a mass-adoption library by count, but its real reach
is indirect — most modern Elixir HTTP traffic goes through Mint via Finch/Req.
Maintenance is active (last push July 2026); the 1.x API has been stable since
2020.

## Getting Started

```elixir
# mix.exs
defp deps do
  [{:mint, "~> 1.0"}]
  # add {:castore, "~> 1.0"} if you're on OTP < 25 (CA certificate store)
end
```

```elixir
{:ok, conn} = Mint.HTTP.connect(:https, "httpbin.org", 443)
{:ok, conn, request_ref} = Mint.HTTP.request(conn, "GET", "/", [], nil)

# The socket is in active mode: responses arrive as messages to the
# process that owns the connection. stream/2 parses them.
receive do
  message ->
    {:ok, conn, responses} = Mint.HTTP.stream(conn, message)
    # [{:status, ^request_ref, 200}, {:headers, ^request_ref, [...]},
    #  {:data, ^request_ref, "..."}, {:done, ^request_ref}]
end

Mint.HTTP.close(conn)
```

Every function returns a new `conn`; discarding it and reusing an old one is
the classic beginner bug — the struct carries parser and protocol state.

## Architecture / How It Works

`Mint.HTTP` is a facade over two protocol implementations, `Mint.HTTP1` and
`Mint.HTTP2`. On `:https` connections the protocol is negotiated via ALPN
during the TLS handshake; on `:http` it defaults to HTTP/1.1 unless you force
`protocols: [:http2]`. All three modules expose the same shape of API:
`connect`, `request`, `stream`, `close`, each threading the immutable conn.

The socket runs in active mode (`active: :once`), so TCP/SSL data arrives as
Erlang messages in the mailbox of the controlling process. `stream/2` is the
heart of the library: fed any message, it either returns `:unknown` (not for
this connection) or parses the bytes into response parts tagged with
per-request refs — `:status`, `:headers`, `:data`, `:done`, `:error`. A
passive mode (`mode: :passive` plus `recv/3`) exists for blocking use, and
ownership moves between processes with `controlling_process/2`, mirroring
`:gen_tcp` semantics.

The HTTP/2 side is a full client implementation: one conn multiplexes many
concurrent streams, with HPACK header compression in `hpax`, a library the
maintainers extracted for reuse[^4]. It exposes machinery HTTP/1 doesn't
have: flow-control window queries (`Mint.HTTP2.get_window_size/2`), server
pushes, and ping frames. TLS verification is on by default with proper
hostname checking — historically not a given among Erlang HTTP clients —
using the system CA store on OTP 25+ or the castore package below that.

## Production Notes

**You are the connection pool.** Mint ships none. HTTP/1 needs N conns for N
concurrent requests; HTTP/2 multiplexes on one conn, but then the single
process handling its messages becomes the serialization point. Checkout,
health checks, and reconnects are real work — precisely the code Finch exists
to own[^2]. Direct Mint use in application code usually means you should be
using Finch or Req.

**Active mode couples the conn to a process.** Response messages land in the
mailbox of whichever process opened (or was given) the connection. Wrapping a
conn in a GenServer and forwarding via `handle_info` → `stream/2` is the
standard pattern; forgetting `:unknown` handling or dropping the updated conn
corrupts parser state in ways that surface as confusing later errors.

**HTTP/2 flow control is your problem.** Sending a body larger than the
current send window errors rather than transparently queueing; you must chunk
against `get_window_size/2` and resume after window updates. Libraries above
Mint hide this; raw users hit it with large uploads.

**Timeouts are manual.** Beyond the connect timeout there is no request
timeout option; in active mode you implement deadlines yourself
(`Process.send_after/3`, `receive ... after`).

**Scope gaps are permanent, not pending.** No redirects, cookies, retries,
compression negotiation, or caching — evaluate Mint as a protocol layer, not
a batteries-included client.

## When to Use / When Not

**Use when:**
- You are building an HTTP-consuming library (SDK, pool, framework) and want
  full control over processes, pooling, and backpressure.
- You need HTTP/2 specifics — multiplexing, server push, flow-control
  introspection — from Elixir.
- You have an architecture where one process should own many sockets (or the
  reverse), which process-per-connection clients can't express.

**Avoid when:**
- You just need to call an API from application code — use Req (or Finch
  directly); both are Mint underneath with the missing 80% filled in.
- You expect a synchronous `get(url)` that returns a response; Mint's
  message-driven API will feel like implementing an HTTP client, because it is.
- You need WebSockets or HTTP/3 — mint_web_socket covers the former on top of
  Mint; HTTP/3/QUIC is not in scope.

## Alternatives

- wojtekmach/req — batteries-included client on Finch/Mint; use for virtually
  all application-level HTTP in Elixir.
- sneako/finch — pooling, telemetry, and a request API on Mint; use when you
  want performance-focused plumbing without Req's conveniences.
- benoitc/hackney — veteran process-based Erlang client with a sync API
  (backs HTTPoison), but historically laxer TLS defaults.
- ninenines/gun — Erlang HTTP/1, HTTP/2, and WebSocket client from the Cowboy
  author; use for always-on connections when process ownership fits.
- erlang/otp `httpc` — stdlib client; fine for zero-dependency scripts, weak
  production defaults.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2019-02 | Public announcement on the Elixir blog[^1]. |
| 1.0.0 | 2020-03 | API declared stable[^5]. |
| 1.x | 2020– | Incremental releases: proxy handling, OTP 25 system CA store support, HPACK extracted to hpax[^4]; no breaking 2.0. |

## References

[^1]: Andrea Leopardi, "Mint, a new HTTP library for Elixir" — Elixir blog, 2019-02-25. https://elixir-lang.org/blog/2019/02/25/mint-a-new-http-library-for-elixir/
[^2]: Finch — HTTP client on Mint + NimblePool. https://github.com/sneako/finch
[^3]: mint_web_socket — WebSocket support over Mint. https://github.com/elixir-mint/mint_web_socket
[^4]: hpax — HPACK implementation extracted from Mint. https://github.com/elixir-mint/hpax
[^5]: Mint changelog / releases. https://github.com/elixir-mint/mint/releases

## Tags

elixir, erlang, beam, http-client, http2, networking, low-level, library, functional
