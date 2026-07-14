# seanmonstar/reqwest

> The default high-level HTTP client for async Rust — a batteries-included wrapper over hyper and tokio.

[GitHub repo](https://github.com/seanmonstar/reqwest) ·
[Documentation](https://docs.rs/reqwest) ·
[License: Apache-2.0 OR MIT](https://github.com/seanmonstar/reqwest/blob/master/LICENSE-APACHE)

## Overview

reqwest is the client most Rust code reaches for when it needs to make an HTTP request. It is written and maintained by Sean McArthur, the same author as hyper — reqwest is the ergonomic layer sitting on top of hyper's low-level HTTP implementation and the tokio async runtime[^1]. Where hyper hands you connections and frames, reqwest hands you `Client`, `RequestBuilder`, and `Response`, plus opt-in conveniences: JSON (de)serialization, multipart forms, cookie stores, redirect policies, gzip/brotli/deflate/zstd decompression, proxies, and TLS.

The defining tension is scope versus dependency weight. reqwest is "batteries-included," which means enabling a few features pulls in serde, a TLS stack, compression libraries, and a large chunk of the tokio ecosystem. For an application this is exactly what you want; for a library author trying to keep the dependency tree small, it is often too much, which is why lighter alternatives exist. The second recurring tension is that reqwest has never shipped a 1.0: it has lived its entire life on 0.x versions, and minor bumps (0.11 → 0.12 → 0.13) have carried real breaking changes, including a swap of the default TLS backend from native-tls to rustls[^2].

It targets async application code first (`tokio`), offers a `blocking` client for scripts and sync contexts, and also compiles to WebAssembly, where it delegates to the browser's `fetch` API with a reduced feature set.

## Getting Started

```toml
[dependencies]
reqwest = { version = "0.13", features = ["json"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

```rust
use std::collections::HashMap;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Reuse one Client across requests — it holds the connection pool.
    let client = reqwest::Client::new();

    let resp = client
        .get("https://httpbin.org/ip")
        .timeout(std::time::Duration::from_secs(10)) // no default timeout — set one
        .send()
        .await?
        .json::<HashMap<String, String>>()
        .await?;

    println!("{resp:#?}");
    Ok(())
}
```

For a synchronous script, enable the `blocking` feature and use `reqwest::blocking::Client` — but note it cannot be called from inside an async runtime (see Production Notes).

## Architecture / How It Works

reqwest is a facade over a stack of lower-level crates, most by the same author:

1. **hyper** — the actual HTTP/1 and HTTP/2 protocol implementation. reqwest does not parse HTTP itself; it configures hyper.
2. **tokio** — the async runtime. reqwest is runtime-coupled: the async client assumes a tokio reactor is running.
3. **tower** — connection pooling and the service/middleware layering used internally.
4. **http / http-body** — the shared type vocabulary (`Method`, `HeaderMap`, `Uri`, `StatusCode`) that the whole Rust HTTP ecosystem agrees on.
5. **A TLS backend** — rustls by default (pure-Rust), or the OS-native stack (`native-tls`: SChannel on Windows, Secure Transport on macOS, OpenSSL on Linux).

A `Client` owns a connection pool keyed by host. It is internally reference-counted (`Arc`), so cloning a `Client` is cheap and shares the same pool — the intended pattern is to construct one `Client` and clone it around, not to build a new one per request. `RequestBuilder` accumulates method, headers, and body; `.send()` drives it through the pool and returns a `Response` whose body is streamed and can be pulled as bytes, text, JSON, or a `futures` stream.

Almost everything beyond a plain GET is behind a Cargo feature flag: `json`, `cookies`, `multipart`, `stream`, `gzip`/`brotli`/`deflate`/`zstd`, `socks`, `native-tls`, `rustls-tls`. This keeps the default build lean but means "reqwest can't parse JSON" is usually a missing-feature problem, not a bug.

The `blocking` client is a thin wrapper that owns its own single-threaded tokio runtime and blocks on the async client internally. On the `wasm32-unknown-unknown` target the whole hyper/tokio stack is replaced by bindings to the browser `fetch` API, and features like custom TLS, proxies, and the blocking client are unavailable.

## Production Notes

**There is no default timeout.** A `send()` with no `.timeout()` (or client-level `Client::builder().timeout(...)`) can hang indefinitely on a stalled connection. Set both a connect timeout and a request timeout explicitly; this is the single most common production footgun.

**Reuse the `Client`.** Constructing `reqwest::Client::new()` per request discards the connection pool every time, forcing a fresh TCP + TLS handshake and defeating keep-alive. Build one `Client` at startup, clone it into handlers. Building a client is also comparatively expensive (it initializes the TLS backend).

**The blocking client cannot run inside async.** Calling `reqwest::blocking` from within a tokio task panics with "Cannot start a runtime from within a runtime." Use the async client in async code; reserve `blocking` for genuinely synchronous programs and CLI tools.

**TLS backend behavior differs.** Since 0.12 the default is rustls, which does **not** consult the OS certificate store the way native-tls does — it uses a bundled root set (webpki-roots) or, with the right feature, rustls-native-certs. Corporate MITM proxies with custom root CAs, or platforms expecting the system trust store, may need `native-tls` or explicit `Certificate` configuration. Switching backends is a frequent source of "works on my machine, fails in the container" TLS errors.

**Decompression is opt-in.** `gzip`, `brotli`, `deflate`, and `zstd` are separate features; without them reqwest will not transparently decode compressed responses even if the server sends them.

**Proxies are read from the environment by default.** reqwest honors `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` unless you disable it, which can surprise deployments that set those vars for other tools.

**Upgrade pain is real for a 0.x crate.** 0.11 → 0.12 tracked hyper's 1.0 rewrite and changed the default TLS backend; ecosystem crates that re-export reqwest types can force a coordinated bump. Pin the minor version and read the changelog before upgrading.

## When to Use / When Not

**Use when:**
- You are writing an async Rust application on tokio and want a client that "just works" with JSON, cookies, redirects, and TLS.
- You need multipart uploads, streaming bodies, or proxy support without wiring hyper yourself.
- You want one client that also compiles to WASM for browser targets.

**Avoid when:**
- You are a library author trying to minimize the dependency tree — reqwest pulls in a large graph; consider a lighter client or exposing hyper directly.
- You want blocking-only HTTP with no tokio in your build — `ureq` is far leaner.
- You need fine control over connections, HTTP/2 frames, or a custom `tower` service — drop down to hyper.
- You require a 1.0-stable API surface with a long support window; reqwest still makes breaking 0.x changes.

## Alternatives

- hyperium/hyper — the low-level HTTP library reqwest is built on; use when you need direct control over connections, protocol details, or a custom service stack.
- algesten/ureq — blocking, minimal, no async runtime; use when you want simple synchronous requests without pulling in tokio.
- sagebind/isahc — libcurl-backed async client; use when you want curl's mature protocol coverage and behavior.
- http-rs/surf — async-std-ecosystem client; largely unmaintained, relevant mainly to legacy async-std codebases.
- hyperium/tonic — gRPC over HTTP/2; use when your protocol is gRPC rather than REST.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9 | 2018-2019 | Last of the synchronous-first line, on old hyper 0.12. |
| 0.10 | 2020-01 | Rewrite onto `async`/`await`, std futures, tokio 0.2, hyper 0.13[^3]. |
| 0.11 | 2021-01 | tokio 1.0, hyper 0.14; long-lived stable line. |
| 0.12 | 2024-03 | hyper 1.0 / http 1.0 migration; default TLS switched to rustls[^2]. |
| 0.13 | 2026 | Current default; `reqwest = "0.13"` shown in the README. |

## References

[^1]: reqwest documentation and crate description. https://docs.rs/reqwest
[^2]: reqwest 0.12 changelog — hyper 1.0 migration and rustls default. https://github.com/seanmonstar/reqwest/blob/master/CHANGELOG.md
[^3]: reqwest 0.10 announcement — async/await rewrite. https://seanmonstar.com/blog/reqwest-010/

## Tags

rust, http-client, http, async, tokio, hyper, rustls, networking, wasm, rest
