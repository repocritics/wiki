# rwf2/Rocket

> An ergonomics-first async web framework for Rust that leans hard on attribute macros to make routes read like plain function signatures.

[GitHub repo](https://github.com/rwf2/Rocket) ·
[Official website](https://rocket.rs) ·
[License: MIT OR Apache-2.0](https://github.com/rwf2/Rocket#license)

## Overview

Rocket is a Rust web framework created by Sergio Benitez, first released in December 2016[^1]. Its defining bet is developer ergonomics: routes are ordinary functions annotated with attribute macros (`#[get("/<name>")]`), and most of the wiring — path parsing, query/form decoding, auth checks, JSON (de)serialization — is expressed declaratively through the type system via request guards and responders. Code that would be middleware plumbing in other frameworks becomes a function parameter whose type implements the right trait.

The central tension in Rocket's history is nightly-vs-stable and the pace that came with it. Through the 0.4 line (2018), Rocket depended on unstable `rustc` features and could only be built on nightly Rust[^2]. Removing that dependency and adding async I/O required a near-total rewrite, which shipped as 0.5 after an unusually long release-candidate period — the first 0.5 RC landed in mid-2021 and the final 0.5.0 not until late 2023[^3]. For roughly five years the "current" way to run Rocket in production was an RC build. That gap is the most-cited criticism of the project and the reason much of the Rust web ecosystem consolidated around axum in the interim.

Since 0.5, Rocket runs on stable Rust and is built on Tokio and Hyper. The repository moved from `SergioBenitez/Rocket` to the `rwf2` organization (Rocket Web Framework Foundation). At ~25.8k stars it remains one of the best-known Rust web frameworks, though the `master` branch is pushed more often than tagged releases appear.

## Getting Started

```bash
cargo add rocket --features json
```

```rust
#[macro_use] extern crate rocket;

use rocket::serde::{Serialize, json::Json};

#[derive(Serialize)]
#[serde(crate = "rocket::serde")]
struct Greeting { message: String }

#[get("/hello/<name>/<age>")]
fn hello(name: &str, age: u8) -> Json<Greeting> {
    Json(Greeting { message: format!("Hello, {age} year old named {name}!") })
}

#[launch]
fn rocket() -> _ {
    rocket::build().mount("/", routes![hello])
}
```

If `<age>` fails to parse as `u8`, the route simply is not matched and Rocket returns 404 — validation is expressed by the parameter type, not by hand-written checks. The `#[launch]` macro builds the async runtime for you.

## Architecture / How It Works

Rocket's request lifecycle is organized around a few trait-based extension points:

- **Request guards** (`FromRequest`) — types that a route asks for as parameters. Extracting an `ApiKey`, a `User`, or a database connection is done by adding a typed argument; if the guard fails, the route is not invoked. Guards run before the handler and can short-circuit with a status.
- **Data guards** (`FromData`) — consume the request body: `Json<T>`, `Form<T>`, and streaming readers. Form handling is a first-class subsystem with typed field parsing and validation.
- **Responders** (`Responder`) — return types that know how to serialize themselves into an HTTP response. `Json<T>`, `Redirect`, `(Status, T)` tuples, templates, and `Result`/`Option` all implement it.
- **Fairings** — Rocket's middleware analog: callbacks on ignite/liftoff/request/response. This is where CORS, request logging, and global headers live.
- **Managed state** — application state registered with `.manage(value)` and requested in handlers via `&State<T>`. Lookup is by type; registering two values of the same type or forgetting to manage one is a runtime, not compile-time, error.

Configuration goes through **Figment** (also Benitez's library): a `Rocket.toml`, environment variables (`ROCKET_*`), and profiles (`debug`/`release`/custom) are merged into a typed config. Under the hood, 0.5 dispatches on Tokio and speaks HTTP via Hyper, with optional TLS (rustls) and HTTP/2.

The macro layer is the framework's signature and its main coupling story. Routing, URI generation (`uri!`), and guard derivation are all code-generated at compile time from the attribute macros, so a mistake in a route signature surfaces as a macro-expansion error rather than a plain type error, and the generated code is a meaningful share of compile time in a Rocket app.

## Production Notes

**Ecosystem interop is the real cost.** axum and tower-based frameworks share the `tower`/`tower-http` middleware ecosystem; Rocket's fairing model does not. Middleware written for the Tower world (rate limiting, tracing, timeout, compression layers) cannot be dropped into Rocket — you reimplement it as a fairing or find a Rocket-specific crate. The third-party crate pool around Rocket is smaller than around axum or actix-web.

**Release cadence is slow and lumpy.** Between 0.4 (2018) and 0.5 (2023) there was no stable release, only RCs; teams that adopted early ran `0.5.0-rc.N` in production for years. Even post-0.5, tagged releases are infrequent relative to `master` activity, so pinning to a git revision to get a fix is a known pattern. Budget for this if you need a steady patch stream.

**Runtime failures from state and guards.** Because managed state is resolved by type at request time, forgetting `.manage()` for a `&State<T>` a handler expects yields a 500 at runtime, not a build failure. Integration tests with the built-in `local` client are the practical guard against this class of bug.

**Macros and compile times.** Rocket leans heavily on proc-macros; incremental builds are fine but clean builds pay for both the codegen and the Tokio/Hyper dependency tree. Error messages originating inside macro expansion are less legible than ordinary Rust type errors, which raises the floor on the learning curve.

**Async migration.** Handlers are `async fn` since 0.5. Blocking work (synchronous DB drivers, heavy CPU) must be moved off the executor with `rocket::tokio::task::spawn_blocking`, same as any Tokio application; blocking in a handler stalls the worker.

## When to Use / When Not

**Use when:**
- You want a batteries-included framework (forms, JSON, templates, TLS, config, cookies) with minimal boilerplate.
- Ergonomics and readability of route code matter more than composing your own stack.
- You're building a conventional server-rendered or JSON API app and value the guard/responder model.

**Avoid when:**
- You need the Tower/`tower-http` middleware ecosystem or want to share layers with other Tokio services — reach for axum.
- You depend on frequent, predictable releases and fast turnaround on fixes.
- You want maximum à la carte control over the HTTP stack, or benchmarked peak throughput is the deciding factor.

## Alternatives

- tokio-rs/axum — Tower-based, minimal, composable; the current default choice in the Rust web ecosystem. Use instead when you want shared middleware and tight Tokio integration.
- actix/actix-web — mature, high-throughput, large ecosystem. Use when raw performance and a deep crate pool are the priority.
- seanmonstar/warp — filter/combinator style built on Hyper. Use when you prefer composing handlers functionally.
- hyperium/hyper — the low-level HTTP library underneath most of the above. Use when you want to build your own abstractions.
- poem-web/poem — ergonomic and feature-rich with first-class OpenAPI. Use when built-in API-spec generation is a hard requirement.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016-12 | Initial release; nightly-only, macro-based routing[^1]. |
| 0.2 | 2017-02 | Streaming responses, config revamp. |
| 0.3 | 2017-07 | Managed state, fairings, typed URIs. |
| 0.4 | 2018-12 | Last nightly-only line; derive macros, revamped forms/config[^2]. |
| 0.5.0-rc.1 | 2021-06 | First async release candidate (Tokio/Hyper), stable Rust[^3]. |
| 0.5.0 | 2023-11 | Final async release; stable Rust, Figment config, rustls TLS[^3]. |

## References

[^1]: Sergio Benitez, "Rocket: Rust Web Framework" — introductory announcement, December 2016. https://rocket.rs/news/2016-12-06-version-0.1/
[^2]: Rocket 0.4 release notes (nightly-only line). https://rocket.rs/v0.4/news/2018-12-08-version-0.4/
[^3]: Rocket 0.5 announcement (async, stable Rust, final release). https://rocket.rs/news/2023-11-17-version-0.5/

## Tags

rust, web-framework, async, tokio, http, macros, backend, api, server
