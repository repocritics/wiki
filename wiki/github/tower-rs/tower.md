# tower-rs/tower

> `async fn(Request) -> Result<Response, Error>` as a trait — the middleware abstraction underneath most of the Rust networking ecosystem.

[GitHub repo](https://github.com/tower-rs/tower) ·
[Documentation](https://docs.rs/tower) ·
[License: MIT](https://github.com/tower-rs/tower/blob/master/LICENSE)

## Overview

Tower is a library of modular, reusable components for building networking clients and servers in Rust. Its entire premise fits in one type: a `Service` is an `async fn(Request) -> Result<Response, Error>` expressed as a trait, and middleware is a function that wraps one `Service` in another. It is protocol-agnostic but assumes a request/response shape; the README is explicit that purely stream-based protocols are a poor fit[^1].

The defining design decision is that readiness is first-class. Every `Service` exposes `poll_ready` alongside `call`, so a service can signal backpressure ("not ready, apply flow control") before a request is ever handed to it. This makes load-shedding, rate limiting, concurrency caps, and buffering expressible as generic, composable layers rather than per-framework special cases. The cost is ergonomics: the trait predates `async fn` in traits and leans on associated `Future` types plus a `poll_ready`/`call` protocol that is easy to misuse by hand.

In practice almost nobody writes `Service` implementations directly. Tower is consumed transitively through frameworks — hyper, tonic (gRPC), axum, warp, and linkerd's proxy all build on the `Service` trait — so the abstraction is load-bearing for a large share of production Rust networking even though most developers never import the `tower` crate themselves. Tower originated from the Linkerd/Buoyant work led by Carl Lerche and the Tokio team[^2].

## Getting Started

```bash
cargo add tower --features full
cargo add tokio --features full
```

The `full` feature pulls in every middleware; in real projects you enable only the ones you use (`timeout`, `limit`, `retry`, `buffer`, `load-shed`, `util`, …), because each is gated behind its own feature flag.

```rust
use std::time::Duration;
use tower::{service_fn, ServiceBuilder, ServiceExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // A Service is just async fn(Request) -> Result<Response, Error>.
    let inner = service_fn(|req: String| async move {
        Ok::<_, std::convert::Infallible>(format!("handled: {req}"))
    });

    // Stack middleware; layers listed outermost-first.
    let mut svc = ServiceBuilder::new()
        .timeout(Duration::from_secs(5))
        .concurrency_limit(64)
        .service(inner);

    // Must await readiness before calling — this is the backpressure contract.
    let out = svc.ready().await?.call("ping".to_string()).await?;
    println!("{out}");
    Ok(())
}
```

## Architecture / How It Works

Tower is really three crates. Two are tiny foundations, `no_std`-compatible, and rarely change:

- **`tower-service`** — defines the `Service` trait. The whole abstraction is `poll_ready(&mut self, cx) -> Poll<Result<(), Error>>` plus `call(&mut self, req) -> Self::Future`, with associated `Response`, `Error`, and `Future` types.
- **`tower-layer`** — defines `Layer<S>`, a single method `fn layer(&self, inner: S) -> Self::Service`. A layer is a service-to-service transformation; that is the definition of middleware here.

The `tower` crate itself is not `no_std`[^1]. It bundles the batteries: `timeout`, `retry` (driven by a user-supplied `Policy`), `limit` (rate and concurrency), `load_shed`, `buffer`, `balance` (load balancing over a `Discover` set), `hedge`, `filter`, and the `util` combinators (`map_request`, `map_response`, `map_err`, `then`, `boxed`).

`ServiceBuilder` is the ergonomic front door: it stacks `Layer`s in written order (outermost first) and hands back a single composed `Service`. `ServiceExt` adds the async helpers — `ready()`, `oneshot()`, `call_all()` — that most callers actually use.

The `poll_ready`/`call` split is the whole point and the whole difficulty. The contract: a caller must drive `poll_ready` to `Poll::Ready(Ok(()))` **before every** `call`, and a service is allowed to reserve resources (a buffer slot, a rate token, a semaphore permit) at readiness time. This is what makes backpressure composable — a `Buffer` layer, for instance, moves the inner service onto a background task and returns `Pending` when its channel is full. It is also what makes hand-written Tower code fragile.

## Production Notes

The Tower abstraction is excellent and the sharp edges are almost all in the type system and the readiness contract.

- **The clone-then-poll footgun.** Many services must be `Clone` (e.g. to hand one per connection to a task pool). But `poll_ready` reserves capacity on the *specific* value you polled. The classic bug: poll readiness on service `A`, then `call` on a fresh `A.clone()` — the reservation is lost, and under load you get panics or lost permits. The documented fix is to clone first and always poll the exact clone you will call. `ServiceExt::ready`/`oneshot` encode the correct dance for you.
- **Type complexity is real.** A `ServiceBuilder` stack produces a deeply nested generic type, and error types unify toward `BoxError` (`Box<dyn Error + Send + Sync>`). Compiler errors on a mis-stacked builder are notoriously long. The escape hatch is `.boxed()` / `BoxService` to erase the type at module boundaries — at the cost of an allocation and dynamic dispatch per call.
- **Feature-flag surprises.** Middleware live behind features. A missing `retry`/`timeout`/`limit` feature shows up as a method-not-found on `ServiceBuilder`, not an obvious "feature disabled" message. `full` is convenient but pulls in more than most services need.
- **Verbosity vs. `async fn` in traits.** Because Tower predates stable async-fn-in-traits, implementing `Service` by hand means writing a `Future` associated type (often `Pin<Box<dyn Future>>` or a hand-rolled state machine) and threading `poll_ready`. This is deliberate — it is what enables backpressure — but it is why direct implementations are rare and why most people stay in framework-provided services.
- **Version churn.** The 0.4 → 0.5 transition (2024) carried breaking changes across the middleware surface; pin the version and read the changelog before upgrading a stack you did not write.
- **MSRV.** Tower keeps a rolling minimum-supported-Rust-version policy of at least six months behind stable; the README states the current MSRV as 1.64.0[^1]. `tower-service` and `tower-layer` are `no_std`; `tower` is not.

## When to Use / When Not

**Use when:**
- You are building a networking client or server and want retry, timeout, rate-limit, concurrency-limit, and load-shed as reusable, testable layers rather than bespoke code.
- You are writing a protocol crate (HTTP, gRPC, a custom RPC) and want a standard middleware interface your users can extend — interop with hyper/tonic/axum comes for free.
- You need real backpressure semantics, not just a linear handler chain.

**Avoid when:**
- Your protocol is purely stream-based with no request/response framing — the README says Tower is a poor fit[^1].
- You need one or two trivial wrappers around a handler; a plain closure or hand-written function composition is simpler than fighting nested generics.
- You want the modern `async fn`-in-traits ergonomics and are willing to give up composable backpressure to get them.

## Alternatives

- hyperium/hyper — the HTTP transport Tower usually sits on top of; call hyper's service closures directly when you only need a layer or two and want to avoid Tower's type gymnastics.
- tokio-rs/axum — built *on* Tower, not a replacement; if you only wanted Tower to run a web server, Axum gives you the ergonomics and reuses any `tower`/`tower-http` layer without hand-assembling the stack.
- actix/actix-web — a self-contained web framework with its own `Transform` middleware model; choose it when you want an all-in-one stack and do not want the `poll_ready` readiness contract.
- tower-rs/tower-http — not an alternative but the companion you will almost always add: HTTP-specific layers (tracing, CORS, compression, auth, request-id, static file serving) built on the same `Service` trait.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017-07 | Repository created; early work out of the Linkerd/Tokio ecosystem[^2]. |
| 0.1 | 2019 | First `tower` crate releases; `Service`/`Layer` split into `tower-service`/`tower-layer`. |
| 0.3 | 2019-11 | Async/await (std `Future`) alignment after Rust 1.39. |
| 0.4 | 2020-12 | Long-lived stable line; `ServiceBuilder`, buffer/balance/retry maturation. |
| 0.5 | 2024 | Breaking-change release; middleware surface cleanup, updated MSRV baseline. |

## References

[^1]: Tower README and crate docs — Overview, `no_std`, and Supported Rust Versions sections. https://github.com/tower-rs/tower and https://docs.rs/tower
[^2]: Tower project background and design (Linkerd/Tokio origins). https://github.com/tower-rs/tower/tree/master/guides

## Tags

rust, middleware, service-trait, networking, async, backpressure, tokio, http, grpc, composability
