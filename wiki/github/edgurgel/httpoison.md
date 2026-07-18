# edgurgel/httpoison

> The long-serving default HTTP client of the Elixir ecosystem — a thin, ergonomic Elixir wrapper over Erlang's hackney.

[GitHub repo](https://github.com/edgurgel/httpoison) ·
[Hex package](https://hex.pm/packages/httpoison) ·
[License: MIT](https://github.com/edgurgel/httpoison/blob/main/LICENSE)

## Overview

HTTPoison is an HTTP client library for Elixir, started by Eduardo Gurgel in 2013 as a port of HTTPotion (which wrapped ibrowse) onto hackney, benoitc's Erlang HTTP client[^1]. For most of Elixir's first decade it was the de facto answer to "how do I make an HTTP request" — `{:httpoison, "~> x.y"}` appears in an enormous number of tutorials, books, and legacy codebases, and at ~2,300 GitHub stars it remains the most-starred Elixir HTTP client.

Its design is deliberately thin: HTTPoison provides idiomatic Elixir function heads (`get/3`, `post/4`, bang variants), response structs you can pattern-match on, and an overridable `HTTPoison.Base` behaviour for building API clients — and delegates everything hard (connection pooling, TLS, proxies, multipart encoding, chunked transfer) to hackney. That thinness is both the appeal and the defining constraint: HTTPoison inherits hackney's capabilities, defaults, bugs, and limits wholesale, including the absence of HTTP/2 support.

The project is mature and still maintained — v3.0.0 shipped in June 2026 to track hackney 4.0[^2], and the last push was July 2026 — but it is in a conservative maintenance mode rather than active feature development. The ecosystem's center of gravity for new projects has shifted to the Mint-based stack (Finch, and Req on top of it), which is pure Elixir, supports HTTP/2, and integrates telemetry natively. HTTPoison today is best understood as the stable incumbent you already have, not the client you would pick fresh.

## Getting Started

```elixir
# mix.exs — HTTPoison 3.x requires hackney 4.0, i.e. Erlang/OTP 27+ / Elixir 1.17+
def deps do
  [{:httpoison, "~> 3.0"}]
end
```

```elixir
case HTTPoison.get("https://api.example.com/users/1",
       [{"Accept", "application/json"}],
       recv_timeout: 10_000) do
  {:ok, %HTTPoison.Response{status_code: 200, body: body}} ->
    Jason.decode!(body)

  {:ok, %HTTPoison.Response{status_code: 404}} ->
    {:error, :not_found}

  {:error, %HTTPoison.Error{reason: reason}} ->
    {:error, reason}
end
```

Note the two-step error model: a 404 is a *successful* request (`{:ok, response}`); only transport-level failures (`:econnrefused`, `:timeout`, `:nxdomain`) return `{:error, %HTTPoison.Error{}}`. Status-code handling is always your job.

## Architecture / How It Works

HTTPoison is a small facade in front of hackney:

- **`HTTPoison.Base`** — a `use`-able macro module that defines the whole client surface. `HTTPoison` itself is just `use HTTPoison.Base` with no overrides. Your own modules can `use HTTPoison.Base` and override `process_request_url/1`, `process_request_headers/1`, `process_response_body/1`, etc. to build declarative API clients (base URL injection, automatic JSON decoding)[^1]. This pre-dates and is cruder than Tesla's middleware stacks or Req's step pipeline — the hooks are compile-time function overrides, not composable runtime plugins.
- **Response structs** — `%HTTPoison.Response{status_code, body, headers, request_url}`, `%HTTPoison.AsyncResponse{}`, `%HTTPoison.Error{}`. Pattern matching on these is the idiomatic control flow.
- **Async streaming** — pass `stream_to: pid` and hackney delivers `%HTTPoison.AsyncStatus{}`, `%HTTPoison.AsyncHeaders{}`, `%HTTPoison.AsyncChunk{}`, `%HTTPoison.AsyncEnd{}` as plain messages to that process's mailbox. With no backpressure by default, a fast server can flood the receiver; `async: :once` plus explicit `HTTPoison.stream_next/1` is the flow-controlled variant[^1].
- **Everything else is hackney passthrough** — options under the `hackney:` keyword (pools, cookies, proxy, `insecure`) go straight to hackney; `:ssl` options go to the Erlang `ssl` module. Multipart bodies use hackney's tuple format verbatim.

The coupling story is the whole story. Connection pooling is hackney's pool model: named pools, checkout/checkin, keepalive reuse per host. TLS defaults (certificate verification via certifi + ssl_verify_fun) are hackney's. HTTP/1.1-only is hackney's. When hackney 4.0 raised its floor to OTP 27, HTTPoison had to cut a major version (3.0) purely to follow[^2]. HTTPoison cannot fix or feature past its engine.

## Production Notes

- **The v2.0 SSL semantics change is the one to know.** Before 2.0, passing *any* `:ssl` option replaced hackney's default SSL options entirely — so adding an innocuous `certfile` could silently disable certificate verification. Since 2.0, `:ssl` options *merge* into hackney's secure defaults, with `:ssl_override` restoring the old replace behaviour explicitly[^3]. Audit any pre-2.0 code you migrate.
- **Pool exhaustion is the classic failure mode.** hackney's default pool caps concurrent connections (per its own defaults) and slow upstreams hold checkouts; under load you get `:checkout_timeout` / `:connect_timeout` errors that look like network failures but are self-inflicted. Size dedicated pools per upstream with `:hackney_pool.start_pool/2` (or as a supervised `child_spec`) rather than sharing `:default`[^4]. Historically, some hackney versions leaked connections back into pools on timeout — pin recent hackney and monitor pool stats.
- **`recv_timeout` defaults to 5000 ms** — routinely too short for slow third-party APIs and file downloads, and the first knob to turn when you see spurious `:timeout` errors.
- **No transparent gzip.** hackney does not auto-decompress `Content-Encoding: gzip` responses; if you send `Accept-Encoding` yourself you must inflate the body manually (`:zlib.gunzip/1`). A recurring gotcha in `HTTPoison.Base.process_response_body/1` callbacks.
- **No HTTP/2, no built-in telemetry.** Services that require HTTP/2 (some gRPC-adjacent APIs, APNs) need Mint/Finch or gun. Instrumentation means wrapping calls yourself or dropping to `:hackney_trace` for low-level wire logging.
- **Upgrade cliffs are OTP-driven.** 3.x requires OTP 27+/Elixir 1.17+[^2]; teams on older Erlang must stay on 2.2.x. The 1.x → 2.0 jump is the SSL merge change above. Otherwise the API has been remarkably stable since 1.0 (2018) — which is exactly why so much legacy code still runs on it untouched.

## When to Use / When Not

**Use when:**
- You maintain an existing Elixir codebase already on HTTPoison — the API is stable and migration buys little unless you need HTTP/2 or telemetry.
- You want a battle-tested HTTP/1.1 client with a decade of Stack Overflow answers and library integrations behind it.
- A dependency you use (older SDKs, ex_aws adapters) already pulls in hackney anyway, so HTTPoison adds near-zero weight.

**Avoid when:**
- You are starting a new Elixir project — wojtekmach/req is the current community default and strictly more capable for the common cases.
- You need HTTP/2, connection multiplexing, or fine-grained backpressure — hackney offers none of these.
- You want first-class telemetry, retries, and JSON handling without hand-rolling them in `HTTPoison.Base` overrides.

## Alternatives

- wojtekmach/req — batteries-included client on Finch/Mint (auto JSON, retries, compression, telemetry); use it for any new Elixir project.
- sneako/finch — pooled, performance-focused client on Mint with HTTP/2; use it when you control high-throughput service-to-service calls.
- elixir-mint/mint — low-level, processless HTTP/1.1+HTTP/2 client; use it when building your own pooling/streaming abstractions.
- elixir-tesla/tesla — middleware-based client with swappable adapters (including hackney); use it when you want composable request pipelines or adapter portability.
- ninenines/gun — Erlang HTTP/1.1, HTTP/2, and Websocket client; use it when you need persistent connections or Websockets from Erlang/Elixir.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013-09 | Initial release; HTTPotion API ported onto hackney[^1]. |
| 0.8.0 | 2015-11 | Maturing pre-1.0 line; async and multipart support established. |
| 1.0.0 | 2018-01 | API stabilized after ~4.5 years of 0.x[^5]. |
| 1.8.0 | 2021-01 | Last 1.x feature release; long maintenance tail follows. |
| 2.0.0 | 2023-01 | `:ssl` options merge with hackney defaults instead of replacing them; `:ssl_override` added[^3]. |
| 2.2.0 | 2023-11 | Maintenance line for OTP < 27 users. |
| 2.3.0 | 2025-11 | Final 2.x release[^5]. |
| 3.0.0 | 2026-06 | Tracks hackney 4.0; requires Erlang/OTP 27+ / Elixir 1.17+[^2]. |

## References

[^1]: HTTPoison README. https://github.com/edgurgel/httpoison#readme
[^2]: HTTPoison v3.0.0 release (2026-06-14) and README requirements: hackney 4.0, Erlang/OTP 27+, Elixir 1.17+. https://github.com/edgurgel/httpoison/releases/tag/v3.0.0
[^3]: "Merge ssl options" — edgurgel/httpoison PR #466, the motivating change for v2.0. https://github.com/edgurgel/httpoison/pull/466
[^4]: hackney README — connection pools, default pool behaviour. https://github.com/benoitc/hackney#use-the-default-pool
[^5]: HTTPoison releases (version dates). https://github.com/edgurgel/httpoison/releases

## Tags

elixir, erlang, http-client, hackney, networking, api-client, otp, library, http1
