# seanmonstar/warp

> A composable Rust web server framework built on hyper, where routes are assembled from typed `Filter` combinators.

[GitHub repo](https://github.com/seanmonstar/warp) ·
[Official website](https://seanmonstar.com/blog/warp/) ·
[License: MIT](https://github.com/seanmonstar/warp/blob/master/LICENSE)

## Overview

warp is a web server framework for Rust written by Sean McArthur, the author of
hyper and reqwest[^1]. It was first published in 2018 as an async-first
alternative to the frameworks of the time, and it sits directly on top of hyper
and Tokio: warp does not implement HTTP itself, it provides a routing and
extraction layer over hyper's server[^2].

Its defining idea is the `Filter`. A filter is a value that inspects a request
and either extracts data from it or rejects it. Filters compose with `.and()`,
`.or()`, `.map()`, and `.and_then()` to build whole applications from small
pieces. This is elegant on paper and genuinely pleasant for small services, but
it is also warp's central tension: because composition happens in the type
system, a realistic route has an enormous concrete type, and a single mistake
produces some of the most infamous compiler errors in the Rust ecosystem.

warp is stable and still maintained, but its cadence is slow. Version 0.3
shipped in January 2021 and remained the current release for over four years,
with only patch releases, before 0.4 arrived in August 2025[^3]. During that gap
much of the Rust community moved to axum, and axum is now the more common
default for new Tokio web services. warp remains a reasonable choice for
existing codebases and for developers who prefer its combinator style.

## Getting Started

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
warp = { version = "0.4", features = ["server"] }
```

```rust
use warp::Filter;

#[tokio::main]
async fn main() {
    // GET /hello/<name> => "Hello, <name>!"
    let hello = warp::path!("hello" / String)
        .map(|name| format!("Hello, {}!", name));

    warp::serve(hello)
        .run(([127, 0, 0, 1], 3030))
        .await;
}
```

As of 0.4 the server is behind a `server` cargo feature, so the dependency must
opt in explicitly[^3]. The `path!` macro builds a filter that matches the path
segments and extracts the typed captures (`String` here) as a tuple.

## Architecture / How It Works

The whole framework is the `Filter` trait. Every filter has an associated
`Extract` tuple (what it pulls out of the request) and a `Rejection` (why it
declined). Combinators build new filters:

- `.and(other)` — run both, concatenate their extracted tuples.
- `.or(other)` — try `self`; if it rejects, try `other`. This is how routing
  works: a big `route_a.or(route_b).or(route_c)` tree.
- `.map(fn)` / `.and_then(async_fn)` — transform the extracted values into a
  reply, synchronously or asynchronously.
- `.recover(fn)` — turn a rejection into a response at the edge of the tree.

Because each combinator returns a distinct concrete type, the type of a
non-trivial application is deeply nested and effectively unwriteable by hand.
Two consequences follow, and they dominate the day-to-day experience:

1. **Error messages.** A type mismatch several combinators deep is reported
   against the fully expanded filter type, which can be hundreds of characters
   long and mentions internal combinator structs. This is the single most
   common complaint about warp.
2. **Splitting routes across functions.** Returning a filter requires a verbose
   signature like `impl Filter<Extract = (impl Reply,), Error = Rejection> +
   Clone`, or erasing the type with `.boxed()` into a `BoxedFilter` at the cost
   of dynamic dispatch.

State (database pools, config) is not injected by an extractor system as in
other frameworks. The idiom is to clone it into a filter with
`warp::any().map(move || state.clone())` and `.and()` it onto the routes that
need it.

The **rejection** model is warp's other distinctive and tricky piece. A
rejection is not immediately an error response; it is a signal that lets `.or()`
fall through to the next branch. Rejections only become HTTP responses when they
reach an unhandled edge or a `.recover()`. Getting rejection ordering and
recovery right — so that a genuine 400 does not surface as a 404 because a later
branch also rejected — is a well-known source of subtle bugs.

Everything below the filter layer is hyper: HTTP/1 and HTTP/2, connection
management, and the async I/O all come from hyper and Tokio, not from warp[^2].

## Production Notes

- **Compile times and type-checker load.** Large `.or()` route trees are slow to
  compile, and deeply nested chains can push rustc into pathological
  type-checking time. Common mitigations are `.boxed()` on sub-trees to cut the
  type depth and splitting routing into separately compiled modules.
- **Maintenance cadence is slow.** The four-plus year gap between 0.3 (2021) and
  0.4 (2025) meant bug fixes and dependency bumps landed infrequently[^3]. Teams
  that need responsive upstream support have tended to migrate.
- **hyper 1.x migration.** warp long tracked hyper 0.14; the 0.4 line is where
  the newer hyper stack and the `server` feature gate landed. Upgrading an
  existing 0.3 app is not a drop-in bump — audit the feature flags and any code
  touching hyper types directly.
- **Middleware story is thin.** warp predates the broad adoption of `tower` as
  the Rust middleware standard, so cross-cutting concerns (timeouts, auth,
  tracing) are often expressed as bespoke filters rather than reusable tower
  layers, which limits ecosystem reuse.
- **Rejection footgun.** Because rejections drive `.or()` fallthrough, an
  overly broad rejection in one branch can mask the intended error from another.
  Always terminate route trees with an explicit `.recover()` that maps known
  rejection types to responses, and return concrete rejections rather than the
  generic not-found.

## When to Use / When Not

**Use when:**
- You already have a warp codebase and it works — there is no forced reason to
  rewrite.
- You genuinely like the filter-combinator style and are building a small or
  medium service where the type ergonomics stay manageable.
- You want a thin, well-understood layer directly over hyper.

**Avoid when:**
- You are starting a new Tokio web service in 2026 — axum covers the same ground
  with far better compiler errors and a larger, more active ecosystem.
- Your team is not fluent in reading complex Rust type errors.
- You need first-class tower middleware, extractor-based state injection, or
  frequent upstream releases.

## Alternatives

- tokio-rs/axum — the de facto successor; same hyper/tower foundation, but
  extractor-based handlers and readable errors. Use for essentially all new
  Tokio web services.
- actix/actix-web — mature, batteries-included, consistently near the top of
  HTTP benchmarks. Use when you want a full-featured framework and peak
  throughput.
- rwf2/Rocket — attribute-macro routing with a Rails-like feel. Use when you
  want ergonomic, declarative routes over combinator composition.
- hyperium/hyper — the layer warp is built on. Use when you want to build your
  own framework or need minimal, unopinionated HTTP.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018-08-01 | Initial release. Filter system over hyper[^1]. |
| 0.2.0 | 2020-01-16 | Migration to `std::future` / async-await and Tokio 0.2. |
| 0.3.0 | 2021-01-19 | Tokio 1.0 and hyper 0.14; long-lived stable line[^3]. |
| 0.3.7 | 2024-04-05 | Last of the 0.3 patch series. |
| 0.4.0 | 2025-08-05 | New release line; `server` feature gate[^3]. |
| 0.4.3 | 2026-05-04 | Latest release as of this writing. |

## References

[^1]: Sean McArthur, "warp" — announcement and background. https://seanmonstar.com/blog/warp/
[^2]: hyper — the HTTP library warp builds on. https://hyper.rs
[^3]: warp releases on crates.io. https://crates.io/crates/warp/versions

## Tags

rust, http, web-framework, server, async, tokio, hyper, filters, backend, routing
