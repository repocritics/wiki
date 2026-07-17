# hyperium/http

> The shared vocabulary of Rust HTTP — `Request`, `Response`, `HeaderMap`, `Uri`, `StatusCode` and nothing else. No client, no server, no I/O.

[GitHub repo](https://github.com/hyperium/http) ·
[Documentation](https://docs.rs/http) ·
[License: Apache-2.0 OR MIT](https://github.com/hyperium/http/blob/master/LICENSE-APACHE)

## Overview

`http` is a types-only crate: it defines the data structures that represent an HTTP request and response — `Request<T>`, `Response<T>`, `HeaderMap`, `HeaderName`, `HeaderValue`, `Method`, `StatusCode`, `Uri`, `Version`, and `Extensions` — and deliberately implements none of the behavior that moves bytes over a socket[^1]. There is no runtime, no async, no parser for the wire protocol, and no networking. It is maintained under the hyperium organization alongside hyper.

The reason it exists as a standalone crate is interoperability. Because hyper, reqwest, axum, tower, tonic, warp, and the rest of the ecosystem all depend on the same `http` types, a piece of middleware written against `http::Request` composes with any of them. The crate is the lingua franca that lets a `tower::Service` written once run under a hyper server, an axum router, or a reqwest client. You almost never add `http` to a project directly for its own sake — it arrives transitively — but you reach for its types the moment you write runtime-agnostic library code.

The defining tension is versioning. `http` sat at 0.2 for roughly four years, and its types leaked into the public API of nearly every HTTP crate in the language. The 1.0 release (late 2023) was a coordinated breaking change: `http::Request` from 0.2 and from 1.0 are distinct, non-interchangeable types, so a dependency tree that pulls both majors produces two incompatible universes of HTTP types[^2]. Managing that split has been the single largest source of friction in the crate's history.

## Getting Started

```toml
[dependencies]
http = "1"
```

```rust
use http::{Request, Response, StatusCode, Method};

fn main() {
    // Builders accumulate errors; the Result surfaces on `.body()`.
    let request = Request::builder()
        .method(Method::GET)
        .uri("https://www.rust-lang.org/")
        .header("User-Agent", "awesome/1.0")
        .body(())
        .expect("valid request");

    let response = Response::builder()
        .status(StatusCode::MOVED_PERMANENTLY)
        .header("Location", "https://www.rust-lang.org/install.html")
        .body(())
        .expect("valid response");

    assert_eq!(request.method(), Method::GET);
    assert_eq!(response.status(), StatusCode::MOVED_PERMANENTLY);
}
```

The generic body parameter `T` is whatever the consumer wants — `()` when there is no body, `Vec<u8>`, a string, or a streaming body type like `http_body::Body`. `http` itself never inspects it.

## Architecture / How It Works

The crate is a small collection of hand-optimized value types, each chosen to avoid allocation on the common path:

- **`HeaderMap`** — not a `HashMap`. It is a purpose-built multi-map that allows multiple values per header name, treats names case-insensitively, and stores standard header names (`content-type`, `host`, …) as static constants so that the common case allocates nothing[^3]. Insertion order and duplicate keys (e.g. multiple `Set-Cookie`) are preserved.
- **`HeaderValue`** — a byte string, *not* a `String`. HTTP header values are not guaranteed UTF-8, so `HeaderValue` stores raw bytes; `.to_str()` returns a `Result`. This is a frequent surprise.
- **`Uri`** — parses a request-target into scheme / authority / path-and-query components backed by a single buffer with byte ranges, avoiding per-component allocation. It is an HTTP-target parser, not a general URL library: it does no percent-decoding, normalization, or relative-reference resolution.
- **`Method`** — an enum for the standard verbs with inline storage for custom extension methods, so a custom method under a small length bound stays on the stack.
- **`StatusCode`** — a validated `u16` wrapper with `const` constructors and associated constants (`StatusCode::OK`).
- **`Request<T>` / `Response<T>`** — a head plus a generic body. The head (`Parts`) is separable via `into_parts()` / `from_parts()`, which is how the ecosystem swaps body types (e.g. buffering a stream) without reconstructing headers.
- **`Extensions`** — a type-keyed map (`TypeId` → `Box<dyn Any>`) that lets middleware attach arbitrary typed data to a request or response as it passes through a stack. This is the mechanism tower/axum use to carry per-request state.

Everything is `no_std`-friendly at the edges but the crate uses `alloc`/`std`. There is no trait surface to implement and no lifecycle — the types are inert data.

## Production Notes

**The 0.2 / 1.0 split is the main operational hazard.** If crate A exposes `http 0.2` types in its public API and crate B expects `http 1.0`, they do not compose, and the compiler error ("expected `http::Request`, found `http::Request`") is famously confusing because the type names are identical. Cargo will happily link both majors simultaneously. Auditing `cargo tree -i http` to see who still pulls 0.2 is a routine part of upgrading a service. The migration wave (hyper 1.0, reqwest, axum 0.7) landed over 2023–2024 but at staggered dates, so mixed trees were common for over a year.

**`HeaderValue` is bytes.** Any code that assumes header values are UTF-8 and unwraps `.to_str()` will panic on non-conforming (but legal) input. Treat it as a fallible conversion.

**`Uri` is not `url`.** Reaching for `http::Uri` to manipulate, normalize, or percent-decode a URL leads to reimplementing a URL library badly. Use the `url` crate (servo/rust-url) for anything beyond parsing an HTTP request-target.

**Builders defer validation.** `Request::builder().uri(bad).header(bad).body(())` does not fail at each call; the first error is stored and returned by `.body()`. Chaining `.unwrap()` on `.body()` turns a malformed header name into a panic, so validate untrusted input before feeding it to the builder.

**MSRV.** The crate follows hyper's MSRV policy and is currently pinned to Rust 1.57[^4], older than most of the ecosystem — convenient for conservative dependents, but it constrains contributions from using newer language features.

**Performance is rarely the bottleneck here.** `HeaderMap` and `Uri` are already tuned; the allocation you should worry about in an HTTP service is almost always in the body/runtime layer (hyper, TLS, serialization), not in constructing `http` types.

## When to Use / When Not

**Use when:**
- You are writing runtime-agnostic library or middleware code and want to interoperate with the whole Rust HTTP ecosystem — depend on `http` types directly.
- You need to represent an HTTP message without committing to a client or server implementation.
- You are building a `tower::Service` or protocol adapter and want types that hyper, axum, and reqwest all already speak.

**Avoid when:**
- You want to actually make requests or serve responses — this crate does nothing on its own; use reqwest or axum/hyper.
- You need URL parsing, normalization, or manipulation — use the `url` crate.
- You want request/response *body* abstractions (streaming, trailers) — that lives in the separate `http-body` crate.

## Alternatives

- hyperium/hyper — the client/server implementation built on these types; add this when you need actual I/O, not just the vocabulary.
- seanmonstar/reqwest — use this instead when you want to make HTTP requests at an application level rather than model messages.
- tokio-rs/axum — use this when you want a web framework; it re-exports and builds on `http`.
- servo/rust-url — use this (the `url` crate) instead of `Uri` when you need to manipulate or normalize URLs rather than parse an HTTP target.
- http-rs/http-types — an alternative HTTP types crate from the async-std/surf/tide lineage; largely superseded now that the ecosystem standardized on `http`.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2017-06 | First release; initial `Request`/`Response`/`HeaderMap` types. |
| 0.2.0 | 2019-12 | The long-lived pre-1.0 line the ecosystem standardized on[^2]. |
| 1.0.0 | 2023-11 | Stable API; coordinated ecosystem break from 0.2[^2]. |
| 1.x | 2024–2026 | Ongoing point releases under the 1.0 stability guarantee. |

## References

[^1]: `http` crate documentation. https://docs.rs/http
[^2]: Sean McArthur, "http 1.0" release announcement, hyper.rs blog (2023). https://seanmonstar.com/blog/http-1.0/
[^3]: `http::HeaderMap` API documentation. https://docs.rs/http/latest/http/header/struct.HeaderMap.html
[^4]: hyper MSRV policy. https://hyper.rs/contrib/msrv/

## Tags

rust, http, http-types, library, networking, web, hyper, cargo-crate, no-runtime, protocol-types
