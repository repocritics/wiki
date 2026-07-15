# quinn-rs/quinn

> Pure-Rust, async QUIC transport — an IETF-spec implementation split into a sans-I/O state machine and a Tokio-based high-level API.

[GitHub repo](https://github.com/quinn-rs/quinn) ·
[Guide book](https://quinn-rs.github.io/quinn/) ·
[License: MIT OR Apache-2.0](https://github.com/quinn-rs/quinn/blob/main/LICENSE-APACHE)

## Overview

Quinn is an implementation of the IETF QUIC transport protocol (RFC 9000 and
related RFCs) written entirely in Rust, with no C QUIC library underneath[^1].
It was started in 2018 by Dirkjan Ochtman and Benjamin Saunders as a side
project, predating the finalized standard, and has tracked the spec through
standardization and past 30 releases since[^2]. QUIC itself is a UDP-based
transport that folds TLS 1.3, stream multiplexing without head-of-line
blocking, and connection migration into one protocol; it is the substrate of
HTTP/3. Quinn implements the transport layer only — HTTP/3 lives in a separate
crate (`h3`).

The defining architectural choice is the split between `quinn-proto` and
`quinn`. `quinn-proto` is a deterministic, sans-I/O state machine: you feed it
timestamps and datagrams, it emits datagrams and events, and it never touches a
socket or a clock itself. `quinn` wraps that machine in a Tokio-based async
runtime, `quinn-udp` handles the platform UDP specifics (ECN, GSO/GRO), and
together they form the high-level API most applications use. This layering is
what lets Quinn be embedded in custom event loops, tested with simulated I/O,
and reasoned about deterministically — at the cost of a more involved API than a
socket-shaped library.

Quinn is pre-1.0 (the `quinn` crate is on the 0.11 line as of 2026) but is
widely deployed: it is the QUIC layer under the `iroh` peer-to-peer stack and a
common choice for Rust services that want QUIC without linking a C library.
Cryptography is pluggable, with the standard path backed by `rustls` and either
`aws-lc-rs` or `ring`.

## Getting Started

```sh
cargo add quinn tokio --features tokio/full
```

A minimal client that opens a bidirectional stream:

```rust
use quinn::{ClientConfig, Endpoint};
use std::net::SocketAddr;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // One endpoint == one UDP socket, regardless of connection count.
    let mut endpoint = Endpoint::client("0.0.0.0:0".parse()?)?;
    endpoint.set_default_client_config(ClientConfig::with_platform_verifier());

    let addr: SocketAddr = "127.0.0.1:4433".parse()?;
    let conn = endpoint.connect(addr, "localhost")?.await?;

    let (mut send, mut recv) = conn.open_bi().await?;
    send.write_all(b"GET /\r\n").await?;
    send.finish()?;                     // finish() is synchronous in 0.11

    let body = recv.read_to_end(64 * 1024).await?;
    println!("{}", String::from_utf8_lossy(&body));
    Ok(())
}
```

The repo ships runnable `server`/`client` examples that generate and trust a
self-signed certificate on the loopback address[^2].

## Architecture / How It Works

The workspace is several crates with a deliberate dependency direction[^2]:

- **`quinn-proto`** — the protocol state machine. No I/O, no async, no timers of
  its own. You drive it: `handle_timeout(now)`, feed inbound datagrams, and poll
  for transmits and application events. This is the crate to depend on for a C
  FFI layer or a non-Tokio runtime.
- **`quinn`** — the async front end. Owns the Tokio tasks that pump the endpoint
  socket, dispatch to per-connection state machines, and expose `Endpoint`,
  `Connection`, `SendStream`, `RecvStream`, and datagram APIs as futures.
- **`quinn-udp`** — the UDP portability layer. Uses `sendmmsg`/`recvmmsg`,
  segmentation offload (GSO) and receive coalescing (GRO), and ECN codepoints
  where the platform supports them; falls back cleanly where it does not.

TLS 1.3 is not reimplemented — it is delegated to `rustls`, which in turn uses a
crypto provider (`aws-lc-rs` by default in recent versions, `ring` as an
alternative). The crypto boundary is a trait, so alternative TLS backends can be
plugged in. Congestion control is likewise pluggable: NewReno, CUBIC, and BBR
implementations ship in-tree, selected per-connection through `TransportConfig`.

Because `quinn-proto` is fully deterministic, the test suite runs connections in
simulated time with no real sockets and no sleeps, which makes timing-sensitive
behavior (loss recovery, timers, ACK logic) reproducible. Setting
`SSLKEYLOGFILE` makes the tests emit real packets and NSS keylogs for Wireshark
inspection[^2].

## Production Notes

**One endpoint is one UDP socket.** Every connection on an `Endpoint` shares a
single socket. At high aggregate throughput the OS default UDP buffers become
the bottleneck and show up as erratic latency or throughput on an otherwise
stable link. The fix is to raise `SO_SNDBUF`/`SO_RCVBUF` before handing the
socket to Quinn — and on Linux that can require elevated privileges or sysctl
changes (`net.core.rmem_max`, `net.core.wmem_max`)[^2].

**Kernel offload matters.** GSO/GRO via `quinn-udp` is a large throughput win on
Linux; on platforms or kernels without it, per-packet syscall overhead
dominates and numbers drop accordingly. Benchmark on the target OS, not just on
Linux.

**Certificate validation is on by default and often awkward for non-web use.**
For peer-to-peer, trust-on-first-use, or servers not named by domain, the
default web-PKI validation does not fit; you customize the `rustls`
`ClientConfig` (see the `insecure_connection` example) rather than reaching for
a Quinn flag. Servers doing TOFU should persist their generated self-signed cert
and reuse it across restarts.

**Crypto backend churn.** The default provider shifted from `ring` toward
`aws-lc-rs`, and `rustls` major versions (0.21 → 0.23) are pinned to specific
`quinn` releases. Mixing a `rustls` version that does not match your `quinn`
version is a common build break; upgrade them together.

**MSRV policy.** The minimum supported Rust version for a published release is
guaranteed to be at least six months old at release time[^2] — conservative,
but it still moves, so pin toolchains in CI if you need reproducibility. Current
MSRV is 1.80.

**Pre-1.0 API.** Minor version bumps (0.10 → 0.11) carry breaking API changes.
Expect to touch endpoint setup and stream-finish call sites on upgrades; the
transport wire protocol is stable (RFC-defined), the Rust API is not.

## When to Use / When Not

**Use when:**
- You want QUIC or HTTP/3 in Rust without linking a C library.
- You need a deterministic, testable transport core (`quinn-proto`) for a custom
  event loop, a non-Tokio runtime, or an FFI surface.
- You're building peer-to-peer or migration-heavy networking where QUIC's
  connection IDs and path migration are the point.

**Avoid when:**
- You need a stable, C-ABI QUIC library to link into an existing C/C++ or nginx
  stack — a C implementation fits better.
- You want a plain reliable byte stream and QUIC's complexity (cert config, UDP
  buffer tuning, pluggable crypto) buys you nothing over TCP+TLS.
- You require a frozen public API; the pre-1.0 crate still breaks across minors.

## Alternatives

- cloudflare/quiche — C/Rust QUIC + HTTP/3 with a C API and BoringSSL; use it when you need to link into C/C++ or nginx-style servers.
- aws/s2n-quic — Rust QUIC from AWS built on `s2n-tls`; use it when you want an AWS-maintained stack and its I/O provider model.
- microsoft/msquic — C QUIC used by Windows and .NET; use it when you need a cross-platform C ABI or Windows kernel-mode support.
- ngtcp2/ngtcp2 — C QUIC library decoupled from any TLS stack; use it when you want to bring your own TLS and integrate at the C level.
- hyperium/h3 — HTTP/3 on top of a QUIC transport (works with Quinn); reach for it when you need HTTP/3 semantics rather than raw streams.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018 | Project founded; RustFest Paris talk, pre-standard[^2]. |
| RFC 9000 | 2021-05 | IETF standardizes QUIC; Quinn tracks the final spec[^1]. |
| 0.9 | 2022 | `rustls`-based crypto, workspace crate split matured. |
| 0.10 | 2023 | `rustls` 0.21 line. |
| 0.11 | 2024 | `rustls` 0.23; `aws-lc-rs` as default crypto provider. |

## References

[^1]: IETF QUIC Working Group, RFC 9000 "QUIC: A UDP-Based Multiplexed and Secure Transport". https://www.rfc-editor.org/rfc/rfc9000
[^2]: quinn-rs/quinn README and guide book. https://github.com/quinn-rs/quinn and https://quinn-rs.github.io/quinn/

## Tags

rust, quic, http3, networking, transport-protocol, async, tokio, tls, udp, sans-io, protocol
