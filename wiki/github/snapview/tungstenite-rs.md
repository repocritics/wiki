# snapview/tungstenite-rs

> A synchronous, stream-based WebSocket (RFC 6455) implementation for Rust — the protocol core that most of the ecosystem's async WebSocket crates are built on.

[GitHub repo](https://github.com/snapview/tungstenite-rs) ·
[docs.rs](https://docs.rs/tungstenite) ·
[crates.io](https://crates.io/crates/tungstenite) ·
License: MIT OR Apache-2.0

## Overview

Tungstenite is a WebSocket protocol implementation that operates over any type implementing `Read + Write` — a `TcpStream`, a TLS stream, an in-memory buffer, anything. It deliberately does not own an event loop, spawn threads, or pick an async runtime. It parses and serializes WebSocket frames, drives the opening/closing handshake as a state machine, and hands you a `WebSocket<S>` you drive yourself[^1]. The name is a pun: it was formerly "WS2" (the second Rust WS implementation), and WS₂ is the chemical formula for tungsten disulfide, the tungstenite mineral[^2].

The defining design choice is that Tungstenite is a *sans-I/O-flavored* core, not a batteries-included server. It is synchronous by nature and integrates with non-blocking sockets by surfacing `WouldBlock` back to the caller, which is what lets it plug into external event loops like MIO. If you want idiomatic async/await, you almost never use this crate directly — you use snapview's own [tokio-tungstenite](https://github.com/snapview/tokio-tungstenite), which wraps this core in Tokio's `AsyncRead`/`AsyncWrite`[^3]. The README says this explicitly: Tungstenite is "more like a barebone," and points async users elsewhere.

That layering is the whole story. Tungstenite is small, maintained (last pushed 2026-07-11), and widely depended-on precisely because it stays narrow: ~2.4k stars on the core crate understate its reach, since most usage arrives transitively through tokio-tungstenite, async-tungstenite, and the WebSocket support in higher-level frameworks.

## Getting Started

```toml
# Cargo.toml — no TLS feature is on by default; add one to reach wss:// endpoints
[dependencies]
tungstenite = { version = "0.24", features = ["rustls-tls-webpki-roots"] }
```

```rust
use std::net::TcpListener;
use std::thread::spawn;
use tungstenite::accept;

// A blocking, thread-per-connection echo server.
fn main() {
    let server = TcpListener::bind("127.0.0.1:9001").unwrap();
    for stream in server.incoming() {
        spawn(move || {
            let mut ws = accept(stream.unwrap()).unwrap();
            loop {
                let msg = ws.read().unwrap();
                if msg.is_binary() || msg.is_text() {
                    ws.send(msg).unwrap();   // send() = write() + flush()
                }
            }
        });
    }
}
```

Client side is symmetric: `tungstenite::connect("wss://…")` performs the handshake and returns the socket plus the HTTP response.

## Architecture / How It Works

The public surface is small and honest about the protocol:

- **`WebSocket<Stream>`** — the driver. `read()` returns the next `Message`; `write()` enqueues one into an internal send buffer; `flush()` pushes that buffer to the underlying stream; `send()` is `write()` + `flush()` for convenience. The split exists so you can batch many `write()`s and flush once.
- **`Message`** — an enum of `Text`, `Binary`, `Ping`, `Pong`, `Close(Option<CloseFrame>)`, and a raw `Frame`. Ping/Pong control frames are surfaced to you (the echo example filters them out manually); auto-pong replies are handled during `read()`/`write()`.
- **Handshake as a state machine** — `accept()` and `connect()` are conveniences over `ClientHandshake`/`ServerHandshake`. On a non-blocking stream a handshake can't finish in one call, so it returns `HandshakeError::Interrupted(MidHandshake)`, which you retry. This is the part people find surprising coming from blocking-only mental models.
- **`WebSocketConfig`** — caps that matter in production: `max_message_size` (default 64 MiB), `max_frame_size` (default 16 MiB), `max_write_buffer_size`, and `accept_unmasked_frames`. These are the backpressure and DoS-mitigation knobs.

TLS is feature-gated and **off by default** — a documented footgun. You opt into exactly one of `native-tls`, `native-tls-vendored`, `rustls-tls-native-roots`, or `rustls-tls-webpki-roots`[^1]. With no TLS feature, `connect("wss://…")` simply cannot reach the endpoint.

Correctness is anchored by the Autobahn Test Suite, the industry conformance suite for WebSocket implementations, which Tungstenite passes, plus internal unit tests[^1]. This is the credible basis for trusting it as a protocol core.

## Production Notes

- **You own concurrency.** A single `WebSocket<S>` is not safe to read and write from two threads simultaneously; the blocking model is one connection per thread (as in the echo example) or a non-blocking socket in your own loop. For async, use tokio-tungstenite instead of trying to bolt async onto this crate.
- **No permessage-deflate.** Compression is not implemented; PRs have been welcomed for years but it is still absent[^1]. If your protocol assumes compressed frames for bandwidth, Tungstenite is not it.
- **The 0.21 API rename bites on upgrades.** The methods were historically `read_message()` / `write_message()`. They were renamed to `read()` / `send()` and the buffered `write()` / `flush()` pair was introduced. Code and tutorials predating that change will not compile against current versions — check which method names your examples use[^4].
- **Frequent minor-version churn.** Tungstenite ships breaking changes on 0.x minor bumps (dependency upgrades to `rustls`, `http`, TLS backends). Because tokio-tungstenite pins a specific Tungstenite range, a security-driven TLS bump often forces a coordinated upgrade across both crates and your framework.
- **Default size caps are generous.** The 64 MiB default `max_message_size` is large for an untrusted public endpoint; lower it explicitly if clients are hostile, or a single peer can make you buffer tens of megabytes.
- **`WouldBlock` is not an error.** On non-blocking sockets, `read()`/`write()` return `Error::Io(WouldBlock)`; treat it as "try again," not a failure, or you will drop live connections.

## When to Use / When Not

**Use when:**
- You need a spec-conformant WebSocket frame layer over a stream you control (TCP, TLS, a test harness, an unusual transport).
- You are building a library or an integration with a non-Tokio event loop (MIO, custom poll loop, embedded-ish).
- You want a thread-per-connection blocking server and value simplicity over per-core scaling.

**Avoid when:**
- You want idiomatic async/await — reach for tokio-tungstenite or async-tungstenite, which wrap this crate.
- You need permessage-deflate compression out of the box.
- You want a full server framework with routing, auth, and connection management — use axum/actix, which provide WebSocket endpoints on top of this stack.

## Alternatives

- snapview/tokio-tungstenite — the async wrapper by the same authors; the default choice for Tokio apps and what most people actually want.
- sdroege/async-tungstenite — runtime-agnostic async wrapper (Tokio, async-std, smol) over the same core.
- denoland/fastwebsockets — performance-focused WebSocket crate; use when Autobahn-grade generality matters less than throughput.
- tokio-rs/axum (and actix/actix-web) — use these when you want a full HTTP framework with a WebSocket upgrade endpoint rather than a raw protocol core.
- housleyjk/ws-rs — older event-loop-based WS crate; effectively unmaintained, listed for historical context only.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017 | Initial release as the successor to the earlier "WS2" implementation[^2]. |
| ~0.15–0.20 | 2021–2023 | `rustls`/`native-tls` feature matrix, `http` crate integration, config caps stabilized. |
| ~0.21 | 2023 | API rename: `read_message`/`write_message` → `read`/`send`, plus buffered `write`/`flush`[^4]. |
| 0.24.x | 2024–2026 | Current line; ongoing TLS/dependency upgrades, active maintenance (last push 2026-07). |

Exact per-release dates for the milestones above are approximate; consult the crate's CHANGELOG for authoritative versions.

## References

[^1]: Tungstenite README — features, TLS matrix, Autobahn conformance, no permessage-deflate. https://github.com/snapview/tungstenite-rs
[^2]: Origin of the name (formerly "WS2", tungsten disulfide WS₂), stated in the README's "Why Tungstenite?" section. https://github.com/snapview/tungstenite-rs
[^3]: tokio-tungstenite — the async wrapper recommended by the README for production async use. https://github.com/snapview/tokio-tungstenite
[^4]: API docs on docs.rs (`read`/`write`/`send`/`flush` on `WebSocket`). https://docs.rs/tungstenite

## Tags

rust, websocket, rfc6455, networking, protocol, sans-io, synchronous, tls, low-level, library
