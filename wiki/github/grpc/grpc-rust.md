# grpc/grpc-rust

> Native async/await gRPC for Rust — the `tonic` crate, built on hyper, tokio, tower, and prost.

[GitHub repo](https://github.com/hyperium/tonic) ·
[Documentation](https://docs.rs/tonic) ·
[License: MIT](https://github.com/hyperium/tonic/blob/master/LICENSE)

## Overview

This repository is `tonic`, a gRPC-over-HTTP/2 implementation written in pure Rust with first-class `async`/`await` support[^1]. It is part of the hyperium organization (the maintainers of `hyper`), and its GitHub URL resolves through a `grpc/grpc-rust` redirect. It is the de facto gRPC library for the Rust ecosystem: if a Rust service speaks gRPC, it almost always speaks it through tonic.

Unlike `tikv/grpc-rs`, which wraps the C-core gRPC library, tonic has no C dependency. It is layered entirely on Rust crates — `hyper`/`h2` for HTTP/2 transport, `tokio` for the runtime, `tower` for middleware, and `prost` for Protocol Buffers codegen. That purity is its selling point (single-language build, no linking against a large C library) and the source of its defining tension: tonic inherits the churn of the entire `tokio`/`hyper` stack. Because it has never reached 1.0, each `0.x` minor is free to make breaking changes, and upgrades frequently force a simultaneous bump of `prost`, `hyper`, `tower`, and `http`.

The project is actively maintained[^2]. As of this writing the last released line is the `0.14.x` branch, while `master` is preparing further breaking changes[^1].

## Getting Started

```toml
# Cargo.toml
[dependencies]
tonic = "0.14"
prost = "0.14"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }

[build-dependencies]
tonic-build = "0.14"
```

```rust
// build.rs — codegen runs at build time
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("proto/helloworld.proto")?;
    Ok(())
}
```

```rust
// src/main.rs — a unary "say_hello" server
use tonic::{transport::Server, Request, Response, Status};
use hello_world::greeter_server::{Greeter, GreeterServer};
use hello_world::{HelloReply, HelloRequest};

pub mod hello_world {
    tonic::include_proto!("helloworld"); // package name from the .proto
}

#[derive(Default)]
pub struct MyGreeter;

#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<HelloReply>, Status> {
        let reply = HelloReply {
            message: format!("Hello {}", request.into_inner().name),
        };
        Ok(Response::new(reply))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse()?;
    Server::builder()
        .add_service(GreeterServer::new(MyGreeter))
        .serve(addr)
        .await?;
    Ok(())
}
```

`tonic-build` invokes the `protoc` Protocol Buffers compiler for `compile_protos`, so `protoc` must be installed on the build machine[^3]. MSRV is 1.88[^1].

## Architecture / How It Works

tonic is three cooperating pieces[^1]:

1. **Generic gRPC core** — codec-agnostic and transport-agnostic traits. gRPC is modeled as `tower::Service`s, so requests flow through the same middleware abstraction the rest of the tokio ecosystem uses.
2. **HTTP/2 transport** — `hyper`/`h2`. gRPC requires HTTP/2 with trailers (status is delivered in the trailing metadata), which constrains what infrastructure can sit in front of it.
3. **Codegen** — `tonic-build` reads `.proto` files and emits Rust client/server stubs, using `prost` for message types by default. Generated code is produced into `OUT_DIR` at build time and pulled in with `include_proto!`; it is not checked into the repo unless you pre-generate it.

The four RPC shapes (unary, server-streaming, client-streaming, bidirectional) map onto Rust `Stream`s. Middleware is expressed either as lightweight `Interceptor`s (metadata/auth) or as full `tower::Layer`s stacked on the transport for anything more involved. TLS is provided through `rustls`. Health checking, server reflection, and gRPC well-known types live in separate crates (`tonic-health`, `tonic-reflection`, `tonic-types`) so the core stays small.

The codec is swappable in principle, but in practice nearly everyone uses `prost` protobuf; alternative codecs exist but are lightly trodden paths.

## Production Notes

- **Version churn is the number-one operational cost.** tonic is pre-1.0 and bumps its public API on minor releases. The `hyper` 0.14 → 1.0 migration (around tonic 0.12) was a hard cutover that changed transport types and rippled into `http`/`tower`. Pin exact versions and expect a real migration each time you upgrade tonic, prost, or the tokio stack together.
- **`protoc` in CI/Docker.** `tonic-build` needs the `protoc` binary at build time[^3]. Slim base images and hermetic CI often lack it; installing it (or vendoring generated code) is a recurring setup step.
- **Default message-size cap.** Decoding is limited to 4 MB by default. Large messages or streamed payloads silently fail until you raise `max_decoding_message_size` / `max_encoding_message_size` on the client or server — a common first-week footgun.
- **Load balancing is basic.** Client-side balancing over a static or `Discover`-fed set of endpoints exists via the channel, but there is no built-in xDS or service discovery. Production deployments typically front tonic with a proxy (Envoy, linkerd) or implement their own `Discover`.
- **HTTP/2 all the way through.** Because gRPC relies on HTTP/2 trailers, every hop (load balancers, ingress, proxies) must support h2 end-to-end. HTTP/1.1-terminating proxies break gRPC.
- **Browsers need grpc-web.** Browsers cannot speak raw gRPC (no trailer access). Serving browser clients requires the `tonic-web` layer plus a compatible JS client; it is not automatic.
- **Build times.** Codegen plus the `prost`/`hyper`/`tokio` dependency tree makes cold builds heavy. Pre-generating stubs or caching `OUT_DIR` helps in CI.

## When to Use / When Not

**Use when:**
- You are building Rust services and want gRPC without linking a C library.
- You already live in the tokio/hyper/tower ecosystem and want RPC that composes with `tower` middleware.
- You need bidirectional streaming, TLS, and interop with gRPC clients in other languages.

**Avoid when:**
- You need Rust-to-Rust RPC only and would rather skip protobuf and HTTP/2 (`tarpc` is lighter).
- You require the full gRPC C-core feature set (xDS, built-in balancing policies) with guaranteed cross-language parity.
- You cannot tolerate frequent breaking upgrades or cannot install `protoc` in your build environment.

## Alternatives

- tikv/grpc-rs — `grpcio`, Rust bindings over the gRPC C-core; use when you want the C-core's full feature set and cross-language conformance and accept a C dependency.
- google/tarpc — Rust-native RPC without protobuf or HTTP/2; use for internal Rust-to-Rust services where interop is not a concern.
- capnproto/capnproto-rust — Cap'n Proto RPC with zero-copy serialization and a capability model; use when you want capabilities-based RPC instead of gRPC semantics.
- tokio-rs/prost — the protobuf codegen tonic builds on, standalone; use when you need protobuf message types but bring your own transport.
- grpc/grpc — the canonical C-core gRPC implementation; use as the reference or from languages other than Rust.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2019 | First release under hyperium; async/await gRPC on hyper + prost[^4]. |
| 0.12 | 2024 | Migration to hyper 1.0 / http 1.0; hard transport cutover. |
| 0.14.x | 2025–2026 | Most recently released line per the README[^1]. |
| master | ongoing | Preparing further breaking changes[^1]. |

Version dates for the 0.x line are approximate; tonic has never released a 1.0, and its minor releases are the unit of breaking change.

## References

[^1]: tonic README, hyperium/tonic. https://github.com/hyperium/tonic
[^2]: GitHub repository metadata (last push 2026-07-14; ~12.4k stars, MIT). https://github.com/hyperium/tonic
[^3]: `tonic_build::compile_protos` docs — requires the `protoc` compiler. https://docs.rs/tonic-build/latest/tonic_build/fn.compile_protos.html
[^4]: "Announcing Tonic" — Lucio Franco, 2019. https://luciofran.co/announcing-tonic/

## Tags

rust, grpc, rpc, protobuf, http2, async, tokio, hyper, networking, prost, streaming
