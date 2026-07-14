# actix/actix-web

> A Rust web framework with a thread-per-core Tokio runtime and a long-standing reputation as one of the fastest in TechEmpower's benchmarks.

[GitHub repo](https://github.com/actix/actix-web) ·
[Official website](https://actix.rs) ·
[License: Apache-2.0 OR MIT](https://github.com/actix/actix-web/blob/main/LICENSE-APACHE)

## Overview

Actix Web is an asynchronous HTTP framework for Rust, first published by Nikolay
Kim (`fafhrd91`) in 2017[^1]. It began as a layer over the `actix` actor
framework — hence the name — but the request-handling path was decoupled from
actors years ago; today an ordinary handler is a plain `async fn`, and the actor
runtime is optional (still used for WebSocket sessions via `actix-web-actors`).
It runs on stable Rust, currently requiring rustc 1.88+[^2].

Its defining trait is performance. For most of the framework's life it has sat at
or near the top of the TechEmpower Framework Benchmark[^3], and that positioning
shaped its design: a thread-per-core execution model, aggressive use of
zero-copy `Bytes`, and (historically) hand-written `unsafe` in hot paths. The
`unsafe` heritage is also the source of its most public episode — a January 2020
dispute over soundness that led the original author to briefly withdraw the
project before handing stewardship to a community team[^4]. The framework that
survived that transition is more conservative about `unsafe` but keeps the
performance-first architecture.

The practical tension: Actix Web is fast and feature-complete (HTTP/1 and HTTP/2,
WebSockets, compression, multipart, TLS via Rustls or OpenSSL), but its type
machinery — extractors, the `Transform`/`Service` middleware traits, and the
per-worker application factory — has a steeper learning curve than newer
Tokio-native frameworks, and its middleware API in particular is verbose to
author by hand.

## Getting Started

```toml
[dependencies]
actix-web = "4"
```

```rust
use actix_web::{get, web, App, HttpServer, Responder};

#[get("/hello/{name}")]
async fn greet(name: web::Path<String>) -> impl Responder {
    format!("Hello {name}!")
}

#[actix_web::main] // or #[tokio::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(greet))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

The closure passed to `HttpServer::new` is an application *factory*: it is
invoked once per worker thread, so anything constructed inside it is per-worker,
not shared. Cross-worker shared state must be wrapped in `web::Data<T>` (an
`Arc`) and registered with `.app_data(...)`.

## Architecture / How It Works

**Thread-per-core runtime.** Actix Web runs on `actix-rt`, a thin layer over
Tokio that spins up one worker thread per logical CPU by default, each with its
own current-thread Tokio runtime. Work is not stolen between workers. This is the
opposite of Axum/Tower's default multi-threaded work-stealing scheduler and is
the main reason a single slow handler can starve everything else pinned to its
worker. It also means per-request state does not need to be `Send`, which is
occasionally convenient and occasionally surprising.

**Extractors.** Handler arguments implement `FromRequest`: `web::Path`,
`web::Query`, `web::Json`, `web::Form`, `web::Data`, `HttpRequest`, and so on.
The handler's return type implements `Responder`. This is ergonomic once
internalized but produces dense, sometimes opaque, compiler errors when a type
doesn't line up — a common early-friction point.

**Middleware.** Middleware is expressed through the Tower-adjacent but
incompatible `Service` / `Transform` trait pair. Writing one by hand means
implementing an associated `Future` type and threading the inner service through
`poll_ready`/`call`. Most users lean on `actix_web::middleware` built-ins
(`Logger`, `Compress`, `NormalizePath`, `DefaultHeaders`) or `wrap_fn` for
simple cases rather than authoring a full `Transform`.

**App composition.** Routing is built from `App` → `Scope` → `Resource` →
`Route`, configurable either with the attribute macros (`#[get(...)]`) or the
builder API (`.route("/", web::get().to(handler))`). Guards allow matching on
method, header, or custom predicates.

**Body & I/O.** Payloads are streamed as `Bytes` chunks; the `awc` client shares
the same HTTP core, so server and client stacks are consistent.

## Production Notes

- **Blocking work blocks a whole worker.** Because each worker is a
  single-threaded runtime, a synchronous or CPU-bound call inside a handler
  stalls every connection on that worker. Move such work to
  `web::block(...)` (a threadpool) or a dedicated executor. This is the single
  most common Actix production footgun.
- **`web::Data<T>` vs `.data()` and per-worker state.** State created in the app
  factory exists once per worker. Anything meant to be shared (a DB pool, a
  cache handle) must be built *outside* the factory and moved in via
  `web::Data`, or you will silently get N independent copies.
- **Panics are isolated but lossy.** A panic in a handler aborts that request's
  worker task; the server keeps running, but you want a `middleware` or panic
  hook plus structured logging, because the default surface is thin.
- **TLS choice matters.** Rustls avoids the OpenSSL system-dependency and
  cross-compilation pain; pick the matching feature flag
  (`rustls-0_23`/`openssl`) deliberately — mixing versions is a frequent build
  error.
- **Graceful shutdown and keep-alive.** `HttpServer` supports graceful shutdown
  with a configurable timeout; long keep-alive or slow-loris style clients are
  handled but the relevant knobs (`keep_alive`, `client_request_timeout`,
  `shutdown_timeout`) are not on by default at the values every deployment
  wants — audit them.
- **Experimental features can break at any release.** Anything prefixed
  `experimental-` (e.g. `experimental-introspection`) carries no semver
  guarantee[^2]; don't build load-bearing infrastructure on it.
- **The 3.x → 4.x jump was large.** Version 4 moved to Tokio 1, std futures, and
  a reworked type surface; upgrading from 3.x was a real migration, not a bump.

## When to Use / When Not

**Use when:**
- Raw throughput and latency are first-order requirements and you want a mature,
  benchmark-proven stack.
- You need a batteries-included HTTP server (WebSockets, multipart, compression,
  TLS, static files) without assembling middleware crates yourself.
- You're comfortable with Rust's async type system and dense trait errors.

**Avoid when:**
- You want the smallest-surface, most idiomatic-Tokio option with the Tower
  middleware ecosystem — Axum fits that better.
- Your workload is CPU-heavy per request and you don't want to reason about
  worker starvation and `web::block`.
- You value a gentle learning curve for a team new to Rust; the extractor and
  middleware machinery has a real ramp.

## Alternatives

- tokio-rs/axum — Tokio/Tower-native, smaller API surface, now the default
  community pick; use it when you want Tower middleware and idiomatic async.
- seanmonstar/warp — filter-combinator style from the hyper author; use it when
  you like composing routes as typed filters.
- SergioBenitez/Rocket — the most "batteries-and-ergonomics" framework; use it
  when developer experience outranks raw benchmark position.
- poem-web/poem — modern, OpenAPI-first design; use it when you want built-in
  spec generation.
- hyperium/hyper — the low-level HTTP implementation under most of the above; use
  it directly when you need full control and no framework opinions.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017-10 | Initial release, built atop the `actix` actor framework[^1]. |
| 1.0 | 2019-06 | First stable line; API stabilization. |
| 2.0 | 2019-12 | Migration to `std::future` and async/await. |
| — | 2020-01 | `unsafe`/soundness dispute; original author steps back, community team takes over[^4]. |
| 3.0 | 2020-09 | Community-maintained release on the 2.x foundations. |
| 4.0 | 2022-02-25 | Tokio 1, std futures, reworked types; major migration from 3.x[^5]. |
| 4.14.0 | 2026 | Current 4.x release; MSRV rustc 1.88+[^2]. |

## References

[^1]: actix/actix-web repository and crate history. https://github.com/actix/actix-web
[^2]: actix/actix-web README — feature list, experimental features, and MSRV (rustc 1.88+). https://github.com/actix/actix-web/blob/main/README.md
[^3]: TechEmpower Framework Benchmarks. https://www.techempower.com/benchmarks/
[^4]: The Register, "Rust-y browser engine Actix ... maintainer quits" (Jan 2020) and community handover discussion. https://www.theregister.com/2020/01/21/rust_actix_web_framework/
[^5]: actix-web 4.0.0 release announcement. https://github.com/actix/actix-web/releases/tag/web-v4.0.0

## Tags

rust, web-framework, http, async, tokio, websockets, backend, http2, server, high-performance
