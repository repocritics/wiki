# hyperium/hyper

> A low-level HTTP/1 and HTTP/2 library for Rust — the plumbing under most of the ecosystem's HTTP clients and servers.

[GitHub repo](https://github.com/hyperium/hyper) ·
[Official website](https://hyper.rs) ·
[License: MIT](https://github.com/hyperium/hyper/blob/master/LICENSE)

## Overview

hyper is an asynchronous HTTP library for Rust that implements the HTTP/1 and
HTTP/2 protocols for both client and server roles[^1]. It has been maintained
primarily by Sean McArthur since 2014 and is one of the oldest and most
depended-upon crates in the async Rust ecosystem. It does not ship a routing
layer, a middleware system, connection pooling, or TLS — it moves bytes and
frames over a byte stream and hands you typed requests and responses built on
the `http` crate.

The defining tension is deliberate. hyper is described by its own README as
"low-level," a building block rather than a batteries-included framework[^1]. If
you want an ergonomic client you are pointed at reqwest; if you want a server
you are pointed at axum or warp[^1]. Almost nobody uses hyper directly — they
use it transitively through those higher-level crates. Choosing to use hyper
raw means accepting responsibility for the runtime glue, the IO adapters, and
the connection lifecycle that the wrappers otherwise hide.

The second defining fact is the 1.0 release (November 2023)[^2], which was a
deliberate stripping-down. Convenience APIs that lived in 0.14 — the pooling
`Client`, the auto-protocol server builder, the Tokio IO glue — were moved out
of the core crate into a companion crate, `hyper-util`. hyper 1.0 froze a small,
stable core and pushed everything opinionated into satellite crates.

## Getting Started

```toml
# Cargo.toml — hyper alone is rarely enough; you almost always add hyper-util + tokio
[dependencies]
hyper = { version = "1", features = ["client", "http1"] }
hyper-util = { version = "0.1", features = ["client-legacy", "http1", "tokio"] }
tokio = { version = "1", features = ["full"] }
http-body-util = "0.1"
```

```rust
use bytes::Bytes;
use http_body_util::Empty;
use hyper::Request;
use hyper_util::client::legacy::Client;
use hyper_util::rt::TokioExecutor;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::builder(TokioExecutor::new()).build_http();
    let req = Request::builder()
        .uri("http://httpbin.org/ip")
        .body(Empty::<Bytes>::new())?;
    let res = client.request(req).await?;
    println!("status: {}", res.status());
    Ok(())
}
```

Note the `hyper-util` types (`Client`, `TokioExecutor`) — in 1.0 those are no
longer part of hyper itself.

## Architecture / How It Works

hyper sits on a stack of small single-purpose crates rather than owning its own
types:

- **`http`** — the shared `Request`, `Response`, `HeaderMap`, `Uri`, `Method`,
  `StatusCode` types. hyper does not define these; the whole ecosystem shares
  them, which is why hyper, reqwest, axum, and tower interoperate.
- **`http-body` / `http-body-util`** — the streaming body trait and adapters
  (`Full`, `Empty`, `BoxBody`, `StreamBody`). In 1.0, bodies are a trait you
  implement, not a concrete type.
- **`h2`** — a separate crate implementing the HTTP/2 framing/state machine.
  hyper's HTTP/1 codec is in-house; HTTP/2 delegates to `h2`.
- **`bytes`** — zero-copy buffer management for the byte plane.

**Runtime abstraction.** hyper 1.0 defines its own `hyper::rt` traits (`Read`,
`Write`, `Timer`, `Executor`) instead of hard-coding Tokio. This is why you
wrap a Tokio socket in `hyper_util::rt::TokioIo` before handing it to hyper —
the core is runtime-agnostic and `hyper-util` supplies the Tokio bindings[^3].
In principle another executor could supply its own bindings; in practice Tokio
is the only well-supported one.

**Service model.** A server is a `hyper::service::Service<Request<B>>` —
a function from request to a future of a response. hyper drives the connection;
your service produces responses. This is the same shape tower's `Service` uses,
which is how tower middleware composes onto hyper-based servers.

**What moved in 1.0.** The 0.14 `hyper::Client` (with connection pooling and
DNS) now lives at `hyper_util::client::legacy::Client`; the auto HTTP/1-or-2
server builder is `hyper_util::server::conn::auto`. The core crate now exposes
mostly connection-level primitives (`hyper::client::conn`, `hyper::server::conn`).

## Production Notes

**The 0.14 → 1.0 migration is the dominant operational story.** It is not a
drop-in upgrade. Code that called `Client::new()` or returned `Body` must be
rewritten against `hyper-util` and `http-body-util`; bodies change from a
concrete `hyper::Body` to a trait-bound generic; IO must be wrapped in
`TokioIo`. Many teams stayed on 0.14 for a long time because the ecosystem
crates (reqwest, axum) migrated on their own schedule[^2]. Check whether your
higher-level crate has already pinned hyper 1 before touching this yourself.

**You probably should not depend on hyper directly.** The most common
production mistake is reaching for raw hyper when reqwest (client) or axum
(server) would do. Raw hyper means you own graceful shutdown, connection
upgrades, keep-alive tuning, and error mapping. The library assumes you know the
HTTP state machine.

**`hyper-util` is pre-1.0 and explicitly not stability-guaranteed.** The core
`hyper` crate is 1.x with semver stability, but the utilities most applications
actually need (`Client`, `auto` server, IO adapters) live in `hyper-util` at
`0.1`. The `legacy` module name signals that even the porting-aid client is not
the intended long-term API.

**No TLS.** hyper speaks cleartext only. HTTPS requires a TLS crate
(`rustls` via `hyper-rustls`, or `native-tls`/`openssl`) wrapping the connection
before it reaches hyper. This is a frequent first-time surprise.

**HTTP/2 tuning matters under load.** Flow-control window sizes, max concurrent
streams, and adaptive windowing are exposed as builder options; defaults are
conservative and high-throughput servers often need them raised. hyper has
historically been a target of CVEs in the shared `h2` dependency around
resource exhaustion (stream/reset floods), so keep `h2` patched.

## When to Use / When Not

**Use when:**
- You are building a higher-level HTTP library, proxy, or framework and want to
  own the connection layer.
- You need precise control over the HTTP/1 or HTTP/2 state machine, upgrades
  (WebSocket, CONNECT), or trailers.
- You are writing a non-Tokio runtime binding and need a runtime-agnostic HTTP
  core.

**Avoid when:**
- You just want to make requests — use reqwest.
- You just want to serve routes — use axum or warp.
- You want TLS, pooling, retries, and timeouts out of the box — those are
  wrapper concerns, not hyper's.
- You cannot absorb the 1.0-era boilerplate (`TokioIo`, `http-body-util`,
  trait-bound bodies).

## Alternatives

- seanmonstar/reqwest — use instead when you want an ergonomic HTTP client; it wraps hyper and adds pooling, TLS, JSON, and redirects.
- tokio-rs/axum — use instead when you want to build a server with routing and extractors on top of hyper + tower.
- seanmonstar/warp — use instead when you prefer a composable, filter-based server API (also hyper-based).
- actix/actix-web — use instead when you want a full, self-contained server framework not built on hyper.
- cloudflare/pingora — use instead when you need a battle-tested proxy framework for very high-scale services rather than a raw HTTP core.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.10 | 2016 | Synchronous, `std::io`-based API (pre-async). |
| 0.11 | 2017 | Rewrite onto futures / async IO. |
| 0.12 | 2018 | Tokio integration. |
| 0.13 | 2019-12 | `async`/`await` support. |
| 0.14 | 2021-01 | Long-lived stable line; `hyper::Client`, `hyper::Body`. |
| 1.0 | 2023-11 | Stable core; convenience APIs split into `hyper-util`[^2]. |

## References

[^1]: hyper README and project site. https://github.com/hyperium/hyper — https://hyper.rs
[^2]: Sean McArthur, "hyper v1.0.0" — 2023-11-15. https://seanmonstar.com/blog/hyper-v1/
[^3]: hyper 1.0 upgrade guide (runtime, IO, body changes). https://hyper.rs/guides/1/upgrading/

## Tags

rust, http, http2, http1, networking, async, tokio, http-client, http-server, low-level, library
