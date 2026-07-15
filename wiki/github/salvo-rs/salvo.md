# salvo-rs/salvo

> A Rust web framework whose central bet is that middleware and handlers are the same thing, so there is almost nothing new to learn past "write a function."

[GitHub repo](https://github.com/salvo-rs/salvo) ·
[Official website](https://salvo.rs) ·
[License: Apache-2.0](https://github.com/salvo-rs/salvo/blob/main/LICENSE)

## Overview

Salvo is an async HTTP web framework for Rust, built on Hyper and Tokio, authored primarily by Chris Learn (chrislearn)[^1]. Its design goal is to minimize the type-level ceremony that other Rust frameworks impose: a request handler is any function annotated with `#[handler]`, and middleware is the same thing — there is no separate `Service`/`Layer`/`Transform` trait hierarchy to implement. This makes Salvo unusually approachable for people who bounce off the trait-heavy surface of axum or the actor model of Actix.

The framework is comparatively young and single-maintainer-centric, which is the tension a prospective adopter should weigh. At ~4.4k stars it is a real project with real users, but it is an order of magnitude smaller than axum or Actix in adoption, contributor count, and third-party middleware. You are trading a gentler learning curve and a batteries-included feature set (built-in OpenAPI, ACME/auto-TLS, HTTP/3, WebTransport) against a smaller ecosystem and a bus factor that is effectively one.

Salvo forbids `unsafe` in its own crate (`#![forbid(unsafe_code)]`)[^2] and tracks a relatively recent minimum Rust version (1.94+ as of mid-2026), so it assumes a current toolchain rather than a conservative MSRV.

## Getting Started

```bash
cargo new hello-salvo
cd hello-salvo
cargo add salvo tokio --features salvo/oapi,tokio/macros
```

```rust
use salvo::prelude::*;

#[handler]
async fn hello() -> &'static str {
    "Hello World"
}

#[tokio::main]
async fn main() {
    let router = Router::new().get(hello);
    let acceptor = TcpListener::new("127.0.0.1:7878").bind().await;
    Server::new(acceptor).serve(router).await;
}
```

Middleware is a handler registered with `hoop`, and it can be scoped to any branch of the routing tree:

```rust
#[handler]
async fn auth_check(depot: &mut Depot, res: &mut Response, ctrl: &mut FlowCtrl) {
    if !authorized(depot) {
        res.status_code(StatusCode::UNAUTHORIZED);
        ctrl.skip_rest();   // short-circuit the remaining handlers
    }
}

let router = Router::new()
    .push(Router::with_path("articles").get(list_articles))
    .push(Router::with_path("articles").hoop(auth_check).post(create_article));
```

## Architecture / How It Works

Salvo sits on top of Hyper for the HTTP protocol machinery and Tokio for the async runtime; it does not reimplement either. Its own contribution is the routing, extraction, and handler model layered on top.

- **Handler unification.** The `#[handler]` macro turns an `async fn` into an implementor of the `Handler` trait. Because middleware and endpoint logic share that one trait, a piece of code can be used as either. Control flow between handlers in a chain is mediated by `FlowCtrl` (`skip_rest`, `call_next`), rather than by a `next()` closure as in tower/express-style stacks.
- **Argument injection.** Handler parameters are resolved by type: a function can ask for `&mut Request`, `&mut Response`, `&mut Depot`, `&mut FlowCtrl`, or any subset, in any order, and the macro wires them up. This is convenient but is macro magic — the coupling between a parameter's type and what gets injected is a convention you have to learn, not something the type system spells out at the call site.
- **Depot** is a per-request typed key/value store used to pass state between middleware and handlers (e.g. an authenticated user set by auth middleware and read downstream). It is Salvo's answer to request extensions.
- **Routing** is a tree (`Router::with_path(...).push(...)`), and filters (path, method, host, custom predicates) attach to nodes. Middleware attached with `hoop` at a node applies to that whole subtree, which is the framework's main composition mechanism.
- **Writer / Scribe.** Response generation goes through the `Writer` trait; returning `Json<T>`, a string, or a custom type works because those implement `Writer`. `Result<T, E>` renders as long as both arms are writable, which is how error handling folds into the same return path.
- **OpenAPI** is first-class: swapping `#[handler]` for `#[endpoint]` (from the `oapi` feature) makes the handler self-describe its parameters and responses, and Salvo generates the spec plus a docs UI (SwaggerUI/Scalar/RapiDoc). This is one of the strongest reasons teams pick Salvo over axum, where OpenAPI is a third-party bolt-on.

Optional protocol and infra features — HTTP/2, HTTP/3 (via Quinn), WebSocket, WebTransport, ACME/Let's Encrypt auto-TLS, rustls/native-tls, compression, CSRF, sessions, rate limiting, proxy, static file serving — are gated behind cargo features on the umbrella `salvo` crate, which re-exports the `salvo-core` and `salvo-extra`/feature crates.

## Production Notes

- **Bus factor.** Development is dominated by a single maintainer. That has produced a coherent, fast-moving codebase, but it is the primary supply-chain and continuity risk for anything mission-critical. Audit the contributor graph and recent release cadence before committing.
- **Breaking changes across minor-ish releases.** Salvo iterated aggressively through its 0.x era and continued to refine APIs after 1.0; `Depot`, extractor, and OpenAPI APIs have shifted between releases. Pin the version and read release notes before upgrading — do not assume semver-patch upgrades are cosmetic.
- **MSRV moves forward.** With a 1.94+ minimum, Salvo will not build on older toolchains; teams with conservative, pinned Rust versions in CI may be blocked. There is no long-tail MSRV support policy comparable to more conservative crates.
- **Macro-driven ergonomics have a debugging cost.** When argument injection or the `#[handler]`/`#[endpoint]` macros go wrong, error messages point into generated code and can be opaque. The convenience that makes Salvo easy to start with makes some failures harder to diagnose than explicit-wiring frameworks.
- **Smaller middleware ecosystem.** Much of what you need ships in-tree (`salvo::prelude` + feature flags), which is good, but for anything outside that set you will more often write it yourself than pull a community crate — the opposite of the tower/axum ecosystem where `tower-http` and friends cover a lot.
- **HTTP/3 and WebTransport are real but bleeding-edge.** They depend on Quinn/QUIC; treat them as advanced features to validate under your own load rather than turnkey production defaults.
- **Benchmarks.** Salvo positions itself as competitive on the TechEmpower and web-frameworks-benchmark boards[^3]. As always, framework microbenchmarks rarely predict your application's bottleneck (DB, serialization, business logic) — verify with your own workload.

## When to Use / When Not

**Use when:**
- You want built-in OpenAPI, auto-TLS/ACME, and HTTP/3 without assembling third-party crates.
- Your team finds axum's trait bounds or Actix's model a barrier and wants a flatter learning curve.
- You are building a service where "middleware is just a handler" maps cleanly onto your needs.

**Avoid when:**
- You need the deepest ecosystem, most Stack Overflow answers, and maximum hiring pool — axum or Actix win on gravity.
- Long-term maintenance guarantees and a large contributor base are a hard requirement (single-maintainer risk).
- You are locked to an older Rust toolchain that cannot meet the MSRV.

## Alternatives

- tokio-rs/axum — the de facto standard; tower-based, larger ecosystem, more explicit types. Use instead when ecosystem gravity and long-term support outweigh ergonomics.
- actix/actix-web — mature, extremely fast, its own actor-influenced model. Use instead when you want a battle-tested framework with years of production history.
- poem-web/poem — closest peer in spirit (ergonomic, built-in OpenAPI). Use instead when you like Salvo's goals but want a different API and community.
- seanmonstar/warp — filter-combinator style on Hyper. Use instead when you prefer composing typed filters over a routing tree.
- rwf2/rocket — batteries-included, macro-heavy, synchronous-feeling ergonomics. Use instead when developer experience and convention matter more than raw async control.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-11-07 | Project started on GitHub[^4]. |
| 0.x | 2020–2024 | Long 0.x era; handler/middleware unification, Depot, routing tree, OpenAPI (`oapi`), ACME, HTTP/3 features added incrementally. |
| Hyper 1.0 migration | ~2024 | Rebased onto the stabilized Hyper 1.x API. |
| 1.0 | 2024 | First stable major release. |
| active | 2026-07 | Ongoing releases; MSRV tracks recent stable Rust (1.94+). |

## References

[^1]: Salvo README and support/credits (Chris Learn / chrislearn). https://github.com/salvo-rs/salvo/blob/main/README.md
[^2]: `unsafe forbidden` badge (rust-secure-code/safety-dance) in the project README. https://github.com/salvo-rs/salvo
[^3]: Performance links cited in the README — Web Frameworks Benchmark and TechEmpower. https://web-frameworks-benchmark.netlify.app/result?l=rust
[^4]: GitHub repository metadata (`created_at` 2019-11-07). https://github.com/salvo-rs/salvo

## Tags

rust, web-framework, http-server, async, tokio, hyper, openapi, http3, websocket, backend
