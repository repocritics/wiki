# JuliaWeb/HTTP.jl

> The HTTP client and server for Julia — the single package nearly every piece
> of Julia web infrastructure sits on.

[GitHub repo](https://github.com/JuliaWeb/HTTP.jl) ·
[Official docs](https://juliaweb.github.io/HTTP.jl/stable/) ·
[License: MIT](https://github.com/JuliaWeb/HTTP.jl/blob/master/LICENSE.md)[^4]

## Overview

HTTP.jl provides HTTP client and server functionality for Julia: requests,
HTTP/2, WebSockets, Server-Sent Events, cookies, multipart forms, retries,
and proxy-aware transports[^1]. It began in December 2016 as the
consolidation of the fragmented early Julia web stack (Requests.jl for
clients, HttpServer.jl for servers) into one JuliaWeb package, with Jacob
Quinn (quinnj) as the long-term primary maintainer.

Its 687 stars understate its position badly. HTTP.jl is infrastructure, not
a destination: Genie, Oxygen.jl, Mux.jl, Pluto, and most registry packages
that touch the network depend on it, so it arrives via `Pkg` dependency
resolution rather than GitHub discovery. Five open issues against a 9+ year
history reflect aggressive triage; the last push was days before writing.

The defining tension is Julia itself: the cooperative task scheduler and JIT
give HTTP.jl a clean task-per-connection model and good steady-state
throughput, but also first-call compile latency and handlers that can starve
the scheduler if they block. The second tension is recency: HTTP.jl 2.0
(May 2026) is a breaking rewrite of the 1.x API, still stabilizing at a
rapid patch cadence[^2].

## Getting Started

```
pkg> add HTTP
```

```julia
using HTTP

r = HTTP.get("http://httpbin.org/ip")
println(r.status)            # 200
println(String(r.body))      # NOTE: aliases the buffer — see Production Notes
```

Server:

```julia
server = HTTP.serve!("127.0.0.1", 8081) do request
    return HTTP.Response(200;
        headers = ["Content-Type" => "text/plain"],
        body = "Hello from HTTP.jl")
end
HTTP.forceclose(server)   # when done
```

Current releases require Julia 1.10+[^1]. There is deliberately no bundled
JSON support — serialize with JSON.jl and set `Content-Type` yourself.

## Architecture / How It Works

The client is a composable **layer stack**: a request passes through handler
layers for redirects, headers, cookies, retries, exception mapping,
connection acquisition, and finally stream I/O. In the 0.x era layers were
encoded at the type level, causing severe compile latency; the 1.0 rewrite
(2022) moved to a runtime stack of plain functions, cutting latency and
allowing user layer insertion. 2.0 reworked the client again around explicit
pooling, timeout, and retry semantics[^2].

The server side is `HTTP.listen!` / `HTTP.serve!`: an accept loop spawning
one Julia `Task` per connection on the cooperative scheduler. `serve!` gives
you a request → response handler; `listen!` and `HTTP.streamhandler` expose
the raw stream for incremental reads, writes, and trailers. Routing beyond a
basic `Router` is left to frameworks (Genie, Oxygen). Parsing is pure Julia
— no C parser dependency — so the package is `Pkg`-installable everywhere
Julia runs. WebSockets live in an `HTTP.WebSockets` submodule with
client/server symmetry. 2.x added first-class HTTP/2, `sse_callback` for
Server-Sent Events, `HTTP.query` for the safe-with-body QUERY method, and a
`ProxyConfig` threaded through client, server, and WebSocket calls[^1].

## Production Notes

**The body-aliasing footgun.** `String(r.body)` takes ownership of the byte
buffer instead of copying — afterwards `r.body` is empty, so reading the
body twice or logging it after parsing breaks silently. Use
`String(copy(r.body))` or stream via `response_stream`[^1]. Documented, but
a recurring source of "empty body" reports.

**The 1.x → 2.0 migration is real work.** 2.0 changed response fields,
constructors, request context, client pooling, retries, timeouts, servers,
WebSockets, and SSE[^2]. If you maintain a package, pin `HTTP = "1"` in
`[compat]` until you migrate deliberately; 1.x received maintenance releases
as recently as March 2026. Six 2.x release series (2.0 → 2.5.5) shipped in
the six weeks after the rewrite[^3] — normal stabilization churn, but pin
exact versions in `Manifest.toml` and read release notes before bumping.

**Scheduler discipline.** A handler doing CPU-bound work or calling a
blocking C library stalls other connections on its thread. Start Julia with
threads (`julia -t auto`) and push heavy work onto worker threads; handlers
are not preemptively scheduled goroutines.

**Latency profile.** The first request after process start pays JIT
compilation; recent Julia precompilation helps but does not eliminate it —
irrelevant for long-lived services, dominant for short-lived processes.

**Serving posture.** A fine application server for APIs, dashboards, and
internal services, but not an nginx-class edge server. Run it behind a
reverse proxy for TLS termination, slow-client buffering, and static files.

## When to Use / When Not

**Use when:**
- You need HTTP in Julia — it is the standard; fighting it costs you the
  ecosystem.
- You want WebSockets, SSE, or streaming bodies in Julia with one dependency.
- You are building or consuming Julia web frameworks (Genie, Oxygen, Mux) —
  they all resolve to HTTP.jl underneath.

**Avoid when:**
- You only need simple file/URL downloads: Downloads.jl ships in the Julia
  stdlib (libcurl-based) with zero added dependencies.
- You need a hardened public-facing edge server — terminate at nginx/Caddy
  and proxy to HTTP.jl.
- Your service is polyglot and Julia is not otherwise required; JIT warmup
  and the deployment story only pay off if you need Julia's numerics.

## Alternatives

- JuliaLang/Downloads.jl — stdlib, libcurl-backed; use for client-only
  fetch/download without a registry dependency.
- OxygenFramework/Oxygen.jl — micro-framework on HTTP.jl; use instead of raw
  handlers when you want routing and middleware.
- GenieFramework/Genie.jl — full-stack MVC framework (also on HTTP.jl); use
  for a batteries-included web app rather than a library.
- JuliaWeb/Mux.jl — thin middleware composition layer; use for Rack/Ring-style
  handler stacks with minimal framework weight.
- JuliaWeb/LibCURL.jl — raw libcurl bindings; use when you need a curl
  feature HTTP.jl's client does not expose.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2016-12 | Repo created; consolidates Requests.jl / HttpServer.jl era. |
| 1.0 | 2022-06 | Stable API; function-based client layer stack replaces type-level layers. |
| 1.11.0 | 2026-03-10 | Final 1.x feature release; 1.x enters maintenance. |
| 2.0.0 | 2026-05-28 | Breaking rewrite: HTTP/2, SSE, QUERY, ProxyConfig; new pooling/retry/timeout semantics[^2]. |
| 2.5.5 | 2026-07-10 | Latest release; rapid post-2.0 stabilization cadence. |

## References

[^1]: HTTP.jl README and documentation. https://juliaweb.github.io/HTTP.jl/stable/
[^2]: HTTP.jl 1.x → 2.0 migration guide. https://juliaweb.github.io/HTTP.jl/dev/guides/migration-1x/
[^3]: Release history. https://github.com/JuliaWeb/HTTP.jl/releases
[^4]: LICENSE.md — MIT "Expat" (GitHub reports "Other" due to a bundled Go-derived cookie-code notice). https://github.com/JuliaWeb/HTTP.jl/blob/master/LICENSE.md

## Tags

julia, http, http-client, http-server, websockets, server-sent-events, http2, networking, web-infrastructure, library
