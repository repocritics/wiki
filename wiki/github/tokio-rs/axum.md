# tokio-rs/axum

> HTTP routing and request-handling library for Rust, built on hyper and tower, with a macro-free API.

[GitHub repo](https://github.com/tokio-rs/axum) ·
[docs.rs](https://docs.rs/axum) ·
[License: MIT](https://github.com/tokio-rs/axum/blob/main/axum/LICENSE)

## Overview

axum is the web framework maintained by the Tokio project. It is deliberately not a full-stack framework: it is a routing and request-handling layer that sits on top of [`hyper`] (the HTTP implementation) and [`tower`] (the middleware/service abstraction). Its defining design decision is that it has **no middleware system of its own** — instead it uses `tower::Service`, so timeouts, tracing, compression, retries, load-shedding, and rate-limiting come from the existing `tower` / `tower-http` ecosystem and are shareable with other `tower`-based stacks like `tonic` (gRPC)[^1].

The ergonomic core is **extractors** and **handlers**. A handler is an ordinary `async fn` whose arguments implement `FromRequest`/`FromRequestParts` (e.g. `Json<T>`, `Path<T>`, `Query<T>`, `State<T>`), and whose return type implements `IntoResponse`. There are no attribute macros on routes — wiring is done with builder calls like `Router::new().route("/users", post(create_user))`. This keeps the API discoverable but pushes most of the complexity into Rust's trait system, which is the source of both its elegance and its worst error messages.

axum is the current default recommendation for new Rust HTTP services precisely because it is the Tokio team's own framework: it tracks `hyper` and `tower` releases closely and inherits their production maturity. The tradeoff is that it is still pre-1.0 — every minor release (`0.6 → 0.7 → 0.8`) is a breaking change, and `main` currently carries in-progress work toward `0.9`[^2].

## Getting Started

```bash
cargo add axum tokio --features tokio/full
cargo add serde --features derive
```

```rust
use axum::{routing::{get, post}, Json, Router, http::StatusCode};
use serde::{Deserialize, Serialize};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(root))
        .route("/users", post(create_user));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn root() -> &'static str {
    "Hello, World!"
}

async fn create_user(Json(payload): Json<CreateUser>) -> (StatusCode, Json<User>) {
    let user = User { id: 1337, username: payload.username };
    (StatusCode::CREATED, Json(user))
}

#[derive(Deserialize)]
struct CreateUser { username: String }

#[derive(Serialize)]
struct User { id: u64, username: String }
```

Note `axum::serve` — since 0.7 axum no longer ships its own server type; you bind a `tokio::net::TcpListener` yourself[^3].

## Architecture / How It Works

The request lifecycle is: `hyper` accepts the connection and produces an `http::Request<Body>` → axum's `Router` matches the path and method → the matched handler's extractors run in argument order to build typed inputs → the handler returns a type implementing `IntoResponse` → the response flows back out through any `tower` layers.

Key internals:

- **Router** — a matcher (backed by `matchit`, a radix-trie router) plus per-route method dispatch. `Router` is itself a `tower::Service`, so routers nest and compose. `.merge()` combines routers; `.nest()` mounts one under a path prefix.
- **Extractors** — `FromRequestParts` extractors (`Path`, `Query`, `State`, headers) only read request metadata and can be used freely; `FromRequest` extractors (`Json`, `Bytes`, `String`, `Form`) **consume the body**. Because a body can only be consumed once, a body extractor must be the *last* handler argument, and only one is allowed. Violating this is a compile-time error, but a cryptic one.
- **State** — application state is threaded via the `State<T>` extractor and `.with_state(value)`. The router is generic over its state type until state is provided, which is why partially-stated routers produce dense generic signatures.
- **Middleware** — applied with `.layer()` (wraps the whole router) or `.route_layer()` (only matched routes). Layers are `tower::Layer`; ordering is outermost-first, which trips up people expecting top-to-bottom application. `middleware::from_fn` lets you write a layer as an async function without implementing `Service` by hand.
- **Handler trait** — `Handler` is implemented for functions of up to 16 extractor arguments via macro-generated impls. When argument types don't line up, the compiler reports "the trait `Handler<_, _>` is not implemented", not the actual mismatch.

axum uses `#![forbid(unsafe_code)]` — the crate itself is 100% safe Rust, delegating all unsafe operations to `hyper`/`tokio`[^4]. Performance is essentially `hyper`'s; axum adds a thin dispatch layer, so it sits near the top of Rust web-framework benchmarks alongside raw `hyper` and `actix-web`[^1].

## Production Notes

**The error messages are the tax.** The single biggest day-to-day friction is that a handler with a wrong argument (a non-extractor type, a missing `Clone` on state, a body extractor in the wrong position) surfaces as a wall of trait-bound errors pointing at the `.route()` call, not the real problem. The standard remedy is `#[axum::debug_handler]` (from `axum-macros`), which rewrites the errors to point at the specific offending argument. Reach for it immediately when a handler "doesn't implement Handler".

**Learning axum is mostly learning tower.** Anything non-trivial — auth middleware, request timeouts, concurrency limits, graceful shutdown, per-route rate limits — is a `tower`/`tower-http` concern. Teams that treat axum as self-contained hit a wall; teams that invest in understanding `Service`, `Layer`, and `ServiceBuilder` ordering get a lot for free.

**Version churn is real.** Because axum is pre-1.0, upgrades are frequent breaking changes, not patch bumps:
- `0.6 → 0.7`: `hyper` upgraded to 1.0; the built-in `axum::Server` was removed in favor of `axum::serve` + a manually-bound `TcpListener`[^3]. Most apps had to rewrite their `main`.
- `0.7 → 0.8`: path parameter syntax changed from `/:id` and `/*rest` to `/{id}` and `/{*rest}`; the `async-trait` crate was dropped in favor of native `async fn` in traits (raising MSRV expectations)[^2]. Silent breakage risk: old-style `:id` routes now match a literal colon.

Pin your minor version and read the CHANGELOG before bumping; the maintainers document migrations well, but the diffs are not mechanical.

**MSRV is 1.80.** Declared minimum supported Rust version[^4]; the native-async-trait move in 0.8 is why it is relatively recent. Compile times are the usual Rust story, aggravated by heavy generic/trait resolution in the router — large route trees noticeably slow incremental builds.

**Graceful shutdown** is not automatic. `axum::serve(...).with_graceful_shutdown(signal)` exists but you must supply the shutdown future (typically a `tokio::signal` handler); forgetting it means in-flight requests are dropped on SIGTERM, which matters under Kubernetes rolling deploys.

## When to Use / When Not

**Use when:**
- You are already on Tokio and want the ecosystem-blessed HTTP layer.
- You want composable, testable services and are willing to learn `tower`.
- You need to share middleware or a runtime with a `tonic` gRPC service.
- You value an explicit, macro-free routing API over convention/attribute magic.

**Avoid when:**
- You want a batteries-included, opinionated full-stack framework (templating, ORM, sessions) out of the box — axum ships none of that.
- Your team is new to Rust and will be blocked by trait-error debugging; a more guided framework may onboard faster.
- You need API stability guarantees now — pre-1.0 breaking releases are ongoing.
- The service is trivial glue where a GC'd language's velocity would win.

## Alternatives

- actix/actix-web — highest raw throughput, larger built-in feature set, its own actor-influenced middleware model instead of tower; use when peak benchmark performance and an all-in-one framework matter more than tower reuse.
- rwf2/Rocket — the most ergonomic, macro-driven Rust framework (attribute routing, request guards); use when developer convenience and a guided API beat composability.
- poem-web/poem — tower-adjacent framework with first-class OpenAPI generation; use when you want auto-generated API specs.
- seanmonstar/warp — axum's filter-based predecessor from the same broader circle; use only for legacy maintenance — new work generally picks axum.
- hyperium/hyper — drop to raw hyper when you want zero framework overhead and are willing to hand-write routing and extraction.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021-07 | Initial release by the Tokio team[^5]. |
| 0.5.0 | 2022-04 | Extractor and error-handling refinements. |
| 0.6.0 | 2022-11 | Typed `State`, compile-time-checked handlers overhaul. |
| 0.7.0 | 2023-11 | hyper 1.0; `axum::Server` removed, `axum::serve` introduced[^3]. |
| 0.8.0 | 2025-01 | Path param syntax `:id` → `{id}`; `async-trait` dropped for native async fn[^2]. |
| 0.9.x | in progress | Breaking changes on `main`; 0.8.x is the released line[^2]. |

## References

[^1]: axum README — "High level features" and "Performance". https://github.com/tokio-rs/axum/blob/main/README.md
[^2]: axum README — "Breaking changes" (working toward 0.9; 0.8.x released) and 0.8.0 CHANGELOG. https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md
[^3]: axum 0.7 announcement — hyper 1.0 and `axum::serve`. https://tokio.rs/blog/2023-11-27-announcing-axum-0-7-0
[^4]: axum README — "Safety" (`#![forbid(unsafe_code)]`) and "Minimum supported Rust version" (1.80). https://github.com/tokio-rs/axum/blob/main/README.md
[^5]: axum announcement post. https://tokio.rs/blog/2021-07-30-announcing-axum
[docs]: https://docs.rs/axum

## Tags

rust, http, web-framework, routing, async, tokio, tower, hyper, backend, api, server
