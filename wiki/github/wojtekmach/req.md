# wojtekmach/req

> A batteries-included HTTP client for Elixir, built as a composable pipeline of request/response "steps."

[GitHub repo](https://github.com/wojtekmach/req) ·
[Official docs](https://hexdocs.pm/req) ·
[License: Apache-2.0](https://github.com/wojtekmach/req/blob/main/LICENSE.md)

## Overview

Req is an HTTP client for Elixir written by Wojtek Mach, first released to Hex in 2021[^1]. It sits at the top of a layered stack: it does not open sockets or manage connection pools itself, but delegates that to Finch (default adapter), which in turn builds on Mint and NimblePool[^2]. Req's own contribution is everything above the wire — automatic body encoding/decoding, redirect following, retries, compression, auth schemes, caching, and testing helpers — packaged so that the common case is one function call with sane defaults.

The defining design choice is that "virtually all features are broken down into individual functions called steps"[^3]. A request is a plain `%Req.Request{}` struct carrying ordered lists of request steps, response steps, and error steps. Making a request means folding the struct through those steps. Because steps are ordinary functions appended to a list, behavior is introspectable and rearrangeable rather than hidden behind adapter middleware macros — the same idea Plug applies to servers, applied to a client.

The main tension for adopters is that Req remains pre-1.0 (the `0.5.x` line as of 2026) despite being widely used and effectively the default recommendation for new Elixir projects. The 0.x versioning signals that option names and step behavior can still change across minor versions, and they periodically have. In practice the library is stable and actively maintained, but the semver contract is weaker than the maturity suggests.

## Getting Started

The fastest path is `Mix.install/2` in an IEx session or script (Elixir v1.12+):

```elixir
Mix.install([{:req, "~> 0.5.0"}])

# GET with automatic JSON decoding of the response body
Req.get!("https://api.github.com/repos/wojtekmach/req").body["description"]
#=> "Req is a batteries-included HTTP client for Elixir."

# POST JSON; :json option encodes the body and sets Content-Type
Req.post!("https://httpbin.org/post", json: %{x: 1, y: 2}).body["json"]
#=> %{"x" => 1, "y" => 2}
```

In a Mix project, add `{:req, "~> 0.5.0"}` to `deps` in `mix.exs`. For repeated calls, build a base request once and reuse it:

```elixir
req = Req.new(base_url: "https://api.github.com")
Req.get!(req, url: "/repos/sneako/finch").body["description"]
```

## Architecture / How It Works

A `Req.Request` struct holds the method, URL, headers, body, options, and three ordered step lists. `Req.request/1` runs them in phases:

1. **Request steps** transform the outgoing struct — `put_base_url`, `encode_body`, `auth`, `compress_body`, `put_params`, `put_path_params`, and so on. The final request step is the adapter (`run_finch` by default), which actually performs the call.
2. **Response steps** transform the result on the way back — `decode_body` (JSON, gzip/brotli/zstd decompression, etc.), `redirect`, `retry`, `checksum`, `cache`.
3. **Error steps** run when the adapter returns an error, primarily to drive retry logic.

Because steps are just `{name, fun}` pairs in a list, you add behavior with `Req.Request.append_request_steps/2` (and the response/error equivalents), and remove or reorder built-ins the same way. Plugins are conventionally packaged as an `attach/1` function that appends steps to a request — `ReqS3`, `ReqHex`, `ReqEasyHTML`, `req_github_oauth`, and others follow this pattern, and `curl_req` converts between Req structs and cURL commands.

The **adapter** is the swap point for the transport. `run_finch` is default, but any function that takes and returns a request can replace it, which is also how `Req.Test` works: it lets you install a stub adapter (or route to a Plug via `run_plug`) so tests never hit the network. Bodies stream in both directions — request bodies via any `Enumerable`, response bodies via `into: fun | collectable | :self`.

The layering is the thing to internalize: Req owns ergonomics and the step pipeline; Finch owns pooling and HTTP/1–HTTP/2 multiplexing; Mint owns the process-less connection state machine. Low-level knobs (pool size, timeouts, TLS/certificates) are not Req options — they are passed through to Finch and documented under `run_finch`[^4].

## Production Notes

- **Pre-1.0 churn.** Pin a tight version (`~> 0.5.0`, not `>= 0.5.0`) and read the CHANGELOG before minor upgrades; option and step semantics have shifted between 0.x minors. This is the single most common upgrade friction.
- **Finch pools are the real tuning surface.** Connection pool sizing, timeouts, and TLS config live in Finch, reached through Req's `finch` / `connect_options` / `pool_timeout` / `receive_timeout` options. Under-provisioned pools surface as queueing latency, not errors, so they are easy to miss until load rises.
- **Retries can amplify load and hide non-idempotent hazards.** The `retry` step is on by default for transient failures; retrying non-idempotent POSTs, or retrying against a struggling upstream, can make an incident worse. Tune `retry`, `max_retries`, and `retry_delay` deliberately.
- **Decoding is automatic and content-type driven.** `decode_body` will parse JSON and decompress gzip/brotli/zstd based on response headers. This is convenient but means a large or malicious response is fully materialized/parsed unless you opt into streaming with `into:` or disable the step.
- **Brotli on macOS needs linker flags.** Building the Brotli NIF locally may require `export LDFLAGS="-undefined dynamic_lookup -dynamiclib"`[^3]; a documented but easy-to-hit dev-environment footgun.
- **Testing is a first-class feature, use it.** `Req.Test` stubs and `run_plug` remove the network from unit tests entirely; prefer them over global HTTP mocks.

## When to Use / When Not

**Use when:**
- You want the default, ergonomic HTTP client for a new Elixir app with JSON decoding, retries, redirects, and auth working out of the box.
- You build API clients and want to package request behavior as reusable steps/plugins.
- You want test stubbing and Plug routing without a separate mocking library.

**Avoid when:**
- You need a hard semver-stable dependency today and cannot tolerate 0.x option churn — vendor a pinned version or weigh a 1.x-tagged alternative.
- You want minimal surface area and full manual control of the connection lifecycle — drop to Finch or Mint directly.
- You are maintaining a legacy codebase already standardized on Tesla or HTTPoison, where migration cost outweighs the ergonomics gain.

## Alternatives

- sneako/finch — the pool/adapter layer Req sits on; use it directly when you want performance and control without the step pipeline.
- elixir-mint/mint — low-level, process-less HTTP/1 and HTTP/2 client; use when you are building your own abstraction.
- elixir-tesla/tesla — middleware-based client with swappable adapters; use when a team is already invested in its middleware model.
- edgurgel/httpoison — older hackney-based client; use only for legacy compatibility, not new code.
- derekkraan/curl_req — companion, not a replacement; converts Req requests to/from cURL commands.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2021-03-28 | Initial commit by Wojtek Mach[^5]. |
| 0.1.0 | 2021 | First Hex release; step-based pipeline established[^1]. |
| 0.3.x | 2022–2023 | Finch consolidated as default adapter; step API matured. |
| 0.4.x | ~2023 | `Req.Test` and Plug-based testing helpers. |
| 0.5.x | 2024–2026 | Current line; AWS SigV4, checksum, expanded streaming and plugins. |

## References

[^1]: Req package on Hex. https://hex.pm/packages/req
[^2]: Finch — Elixir HTTP client focused on performance, the default Req adapter. https://github.com/sneako/finch
[^3]: Req README, "Features", "Development", and step documentation. https://github.com/wojtekmach/req
[^4]: `run_finch` step documentation (lower-level HTTP options: timeouts, pool sizes, certificates). https://hexdocs.pm/req/Req.Steps.html#run_finch/1
[^5]: wojtekmach/req repository metadata (created 2021-03-28). https://github.com/wojtekmach/req

## Tags

elixir, http-client, api-client, networking, finch, mint, functional, batteries-included, library, beam
