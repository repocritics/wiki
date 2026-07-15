# poem-web/poem

> A full-featured async Rust web framework whose distinguishing bet is type-first, derive-generated OpenAPI — the FastAPI experience on hyper/tokio.

[GitHub repo](https://github.com/poem-web/poem) ·
[Crates.io](https://crates.io/crates/poem) ·
[License: Apache-2.0 OR MIT](https://github.com/poem-web/poem/blob/master/LICENSE-APACHE)

## Overview

Poem is an async web framework for Rust built on `hyper` and the `tokio`
runtime[^1]. It sits in the same layer as `axum`, `actix-web`, and `warp`:
routing, extractors, middleware, and a typed request/response model over HTTP.
The repository is a Cargo workspace of several crates — `poem` (the core
framework), `poem-openapi`, `poem-grpc`, `poem-lambda`, and `poem-mcpserver` —
so "Poem" usually means the ecosystem, not just the base crate.

The defining feature is `poem-openapi`: you annotate an `impl` block with the
`#[OpenApi]` macro and Poem generates a conformant OpenAPI 3 specification and a
served Swagger UI / RapiDoc from your Rust types at compile time[^2]. This is a
deliberate parallel to Python's FastAPI (the repo even tags itself `fastapi`),
and it is the main reason to choose Poem over `axum`, which has no first-party
OpenAPI story and leans on third-party crates (`utoipa`, `aide`) for the same
result.

The central tension is ecosystem size versus feature breadth. Poem is
substantially the work of a single primary author, `sunli` — also the author of
`async-graphql` — and its community, third-party middleware, and Stack Overflow
surface area are much smaller than `axum`'s. You trade a larger crowd and the
Tower ecosystem's gravity for a more batteries-included, OpenAPI-native
experience out of one coherent design.

## Getting Started

```bash
cargo add poem
cargo add tokio --features rt-multi-thread,macros
```

```rust
use poem::{get, handler, listener::TcpListener, web::Path, Route, Server};

#[handler]
fn hello(Path(name): Path<String>) -> String {
    format!("hello: {name}")
}

#[tokio::main]
async fn main() -> Result<(), std::io::Error> {
    let app = Route::new().at("/hello/:name", get(hello));
    Server::new(TcpListener::bind("0.0.0.0:3000"))
        .run(app)
        .await
}
```

OpenAPI-first, the framework's actual selling point:

```rust
use poem_openapi::{OpenApi, param::Query, payload::PlainText};

struct Api;

#[OpenApi]
impl Api {
    #[oai(path = "/hello", method = "get")]
    async fn index(&self, name: Query<Option<String>>) -> PlainText<String> {
        match name.0 {
            Some(name) => PlainText(format!("hello, {name}!")),
            None => PlainText("hello!".to_string()),
        }
    }
}
```

`OpenApiService::new(Api, "demo", "1.0")` then yields both the mounted routes
and a `.swagger_ui()` endpoint derived from the same types.

## Architecture / How It Works

The core abstraction is the **`Endpoint` trait** — `async fn call(&self, req:
Request) -> Result<Response>`. A handler is an `Endpoint`; middleware is a
`Middleware` that wraps one `Endpoint` into another. This is Poem's own
abstraction rather than Tower's `Service`, though Poem ships a `TowerCompatExt`
to consume Tower services and layers where you need them[^3]. Choosing a bespoke
`Endpoint` trait (over adopting Tower wholesale like `axum`) is the framework's
most consequential design decision: ergonomics and compile errors are simpler,
but you are partly outside the Tower middleware ecosystem.

Request data is pulled via the **`FromRequest`** extractor pattern — `Path`,
`Query`, `Json`, `Form`, `Data<T>` (shared state), typed headers — matching the
axum/actix mental model. Return values implement **`IntoResponse`**. Errors flow
through a `Result` with a Poem `Error` that carries an HTTP status.

`poem-openapi` is a separate layer built on procedural macros. `ApiResponse`,
`Object`, `Enum`, and `#[OpenApi]` derive both the runtime request/response
handling and the schema emitted into the spec, so the served documentation
cannot drift from the handler signatures. `poem-grpc` generates client and
server code from `.proto` files; `poem-lambda` adapts the `Endpoint` model onto
the AWS Lambda runtime; `poem-mcpserver` exposes handlers as a Model Context
Protocol server. All ride the same `hyper` + `tokio` foundation.

The base crate forbids `unsafe` (the repo carries the `unsafe-forbidden`
badge)[^4] and, at time of writing, requires rustc 1.85.0 or newer.

## Production Notes

- **One-maintainer risk.** Development is healthy — the default branch saw
  commits within roughly two months of this writing — but the bus factor is low
  and issue triage can lag (nearly 200 open issues). Evaluate whether you are
  comfortable depending on a framework whose direction is set by one person.
- **Smaller integration surface than axum.** Auth, observability, and rate-limit
  middleware you would find pre-built for Tower/axum may need the
  `TowerCompatExt` bridge or a hand-rolled `Middleware`. Budget for that.
- **Semver churn across major lines.** Poem has moved through 1.x, 2.x, and 3.x
  major series; upgrades have carried breaking API changes, and `poem` and
  `poem-openapi` versions are coupled — pin them together and read the per-crate
  CHANGELOGs before bumping.
- **MSRV moves forward.** The 1.85.0 floor tracks a recent toolchain; teams on
  pinned older Rust should check `rust-version` before adopting or upgrading.
- **OpenAPI macros dominate build/error complexity.** The `#[OpenApi]` derive is
  where compile times and cryptic macro errors concentrate; a malformed response
  type produces trait-resolution errors that are harder to read than plain
  handler mistakes.
- **Dual-licensed Apache-2.0 OR MIT** — permissive and unambiguous for
  commercial use, despite GitHub reporting only the Apache SPDX.

## When to Use / When Not

**Use when:**
- You want typed, always-in-sync OpenAPI docs generated from your Rust types
  without bolting on a second crate.
- You want a batteries-included async framework (gRPC, Lambda, MCP) from one
  cohesive project.
- You like the FastAPI workflow and want its equivalent with Rust's performance
  and type safety.

**Avoid when:**
- You want the largest community, most third-party middleware, and most hiring
  familiarity — `axum` wins on ecosystem gravity.
- You are deeply invested in the Tower `Service`/`Layer` ecosystem and want it
  as the native, first-class abstraction.
- Single-maintainer dependency risk is unacceptable for your organization.

## Alternatives

- tokio-rs/axum — the default modern choice; Tower-native, larger community, no
  built-in OpenAPI. Use instead when ecosystem size and Tower integration matter
  more than turnkey OpenAPI.
- actix/actix-web — mature, very fast actor-influenced framework. Use when you
  want the longest production track record and top-tier benchmarks.
- seanmonstar/warp — filter-composition framework from the hyper author. Use when
  you prefer a functional, combinator-style API.
- utoipa (with axum) — pair `utoipa` with axum to approximate `poem-openapi`. Use
  when you want OpenAPI but are unwilling to leave the axum ecosystem.
- salvo-rs/salvo — another batteries-included async framework with OpenAPI
  support. Use when comparing full-featured alternatives to Poem directly.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2021-08-11 | Workspace started by `sunli` (author of `async-graphql`)[^1]. |
| 1.x | 2021–2022 | Core `Endpoint`/extractor model; `poem-openapi` established as the differentiator. |
| 2.x | 2024 | Breaking API revision of the core crate. |
| 3.x | 2024–present | Current major line; MSRV advanced to rustc 1.85.0. |

## References

[^1]: poem-web/poem repository and README. https://github.com/poem-web/poem
[^2]: `poem-openapi` crate documentation. https://docs.rs/poem-openapi
[^3]: `poem` crate documentation (`Endpoint`, `Middleware`, Tower compatibility). https://docs.rs/poem
[^4]: Rust Secure Code WG "safety-dance" / unsafe-forbidden badge. https://github.com/rust-secure-code/safety-dance

## Tags

rust, web-framework, async, http, openapi, tokio, hyper, grpc, api, server, backend
