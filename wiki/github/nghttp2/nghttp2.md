# nghttp2/nghttp2

> HTTP/2 implemented as a reusable C library, plus the client, server, proxy, and load-test tools built on top of it.

[GitHub repo](https://github.com/nghttp2/nghttp2) ·
[Official website](https://nghttp2.org) ·
[License: MIT](https://github.com/nghttp2/nghttp2/blob/master/COPYING)

## Overview

nghttp2 is a C implementation of HTTP/2, written by Tatsuhiro Tsujikawa and forked from the earlier SPDY library `spdylay` in 2013[^1]. Its core deliverable is `libnghttp2` — a framing-layer library that speaks the HTTP/2 wire protocol (RFC 7540, now tracking RFC 9113) and exposes an HPACK header-compression encoder/decoder as public API[^2]. It is one of the most widely embedded HTTP/2 stacks in the C/C++ world: curl's HTTP/2 support and Apache httpd's `mod_http2` are both built on libnghttp2, and it ships in most Linux distributions as `libnghttp2`.

The defining design decision is that **libnghttp2 does no I/O**. It never touches a socket, never allocates a TLS session, never runs an event loop. You feed it received bytes, it invokes your callbacks (a header was received, a DATA frame arrived, "send these bytes now"), and you are responsible for the transport. This makes the library transport-agnostic and embeddable inside any existing event loop, at the cost of a steep, callback-heavy integration surface — a first HTTP/2 client is several hundred lines before it does anything useful.

The repository is more than the library. The `src/` tree contains a suite of reference tools that are production software in their own right: `nghttp` (client), `nghttpd` (server), `nghttpx` (a TLS-terminating reverse proxy used in production deployments), and `h2load` (an HTTP/2, HTTP/1.1, and HTTP/3 benchmarking tool that is a de facto standard for load-testing HTTP/2 endpoints). HTTP/3 support in `nghttpx` and `h2load` is provided through the author's sibling projects, ngtcp2 (QUIC) and nghttp3 (HTTP/3)[^3].

## Getting Started

Debian/Ubuntu and most distros ship the library; install the C library and headers directly:

```bash
sudo apt-get install libnghttp2-dev   # library + headers
sudo apt-get install nghttp2-client   # the nghttp / h2load CLI tools
```

Build the library only from a release tarball (avoids the app dependencies):

```bash
./configure --enable-lib-only
make
```

A minimal client submit, showing the callback-driven model (error handling elided):

```c
#include <nghttp2/nghttp2.h>

nghttp2_session *session;
nghttp2_session_callbacks *cbs;

nghttp2_session_callbacks_new(&cbs);
nghttp2_session_callbacks_set_on_data_chunk_recv_callback(cbs, on_data);
nghttp2_session_callbacks_set_on_frame_recv_callback(cbs, on_frame);
nghttp2_session_client_new(&session, cbs, user_data);

const nghttp2_nv hdrs[] = {
    {(uint8_t*)":method",    (uint8_t*)"GET",         7, 3, NGHTTP2_NV_FLAG_NONE},
    {(uint8_t*)":scheme",    (uint8_t*)"https",       7, 5, NGHTTP2_NV_FLAG_NONE},
    {(uint8_t*)":authority", (uint8_t*)"nghttp2.org", 10, 11, NGHTTP2_NV_FLAG_NONE},
    {(uint8_t*)":path",      (uint8_t*)"/",           5, 1, NGHTTP2_NV_FLAG_NONE},
};
nghttp2_submit_request2(session, NULL, hdrs, 4, NULL, NULL);

/* You drive the loop: pull outbound bytes and write them to your socket,
   read inbound bytes and push them in. */
nghttp2_session_send(session);        /* invokes send_callback with framed bytes */
/* ...on socket read... */
nghttp2_session_mem_recv2(session, buf, len);
```

## Architecture / How It Works

libnghttp2 is a **state machine over the HTTP/2 framing layer**. The central object is `nghttp2_session` (one per connection), which owns per-stream state, the HPACK dynamic tables (separate inbound and outbound), and flow-control windows (connection-level and per-stream). You interact through two byte pumps and a set of callbacks:

- `nghttp2_session_mem_recv2()` — hand it received bytes; it parses frames and fires callbacks (`on_begin_headers`, `on_header`, `on_data_chunk_recv`, `on_frame_recv`, `on_stream_close`).
- `nghttp2_session_send()` / `nghttp2_session_mem_send2()` — it serializes queued frames and either calls your `send_callback` or returns the bytes for you to write.

The `2` suffix on `mem_recv2`, `mem_send2`, `submit_request2` etc. marks the current-generation API that uses `nghttp2_ssize` (signed size) return types; the un-suffixed originals are retained for ABI compatibility and are deprecated. New code should use the `*2` variants.

**HPACK** (RFC 7541) is exposed independently via `nghttp2_hd_deflate_*` and `nghttp2_hd_inflate_*`, so the compression codec can be used without the rest of the session machinery. This is a genuine reusable component — several unrelated projects use nghttp2 solely for HPACK.

**Flow control** is the operator's main correctness concern. nghttp2 tracks WINDOW_UPDATE accounting but the application must consume data and call `nghttp2_session_consume()` (or use auto-window-update) or streams stall. Priority handling followed RFC 7540's dependency-tree model; RFC 9113 deprecated that scheme, and the library's newer versions emphasize the simpler default scheduling.

The **tools** compose the library differently. `nghttpx` is the most substantial: a multi-threaded, libev-based reverse proxy that terminates TLS (via OpenSSL, wolfSSL, LibreSSL, aws-lc, or BoringSSL), speaks HTTP/1.1, HTTP/2, and optionally HTTP/3 on the frontend, and can proxy to HTTP/1.1 or HTTP/2 backends. It supports graceful configuration reload and hot executable swap, which on the HTTP/3 path require an eBPF `SO_REUSEPORT` steering program to route QUIC datagrams to the correct worker socket.

## Production Notes

**libnghttp2 has no memory of your I/O intentions.** Two footguns dominate real integrations: (1) forgetting to drive `nghttp2_session_want_write()` after submitting frames, so requests silently never leave; and (2) flow-control deadlock — if you don't consume received DATA and issue window updates, the peer stops sending and the connection hangs with no error. Both are integration bugs, not library bugs, and both are common.

**C++23 for the tools, C99 for the library.** Since the move to modern C++, building the applications (`nghttpx`, `h2load`, etc.) requires g++ >= 14 or clang++ >= 19[^4]. The library itself still only needs a C99 compiler, so `--enable-lib-only` is the right call for embedders who don't want the toolchain burden. The repo also declares the `cpp20`/`cpp23` toolchain expectation in its topics and README.

**Long-running servers fragment the heap.** The README explicitly recommends jemalloc for `nghttpd` and `nghttpx` to mitigate fragmentation in long-lived processes — and notes that Alpine/musl cannot do malloc replacement, so this mitigation is unavailable there (tracked in issue #762)[^5].

**HTTP/3 is a heavier lift.** Enabling `--enable-http3` pulls in ngtcp2 and nghttp3 at specific minimum versions, plus a QUIC-capable TLS library (quictls, aws-lc, BoringSSL, wolfSSL, LibreSSL, or OpenSSL >= 3.5). Version skew between nghttp2, ngtcp2, and nghttp3 is a real source of build friction because the three release on independent cadences and the QUIC/HTTP3 APIs are still evolving.

**Shared-library versioning.** The runtime soname (`libnghttp2.so.14`) is stable, and the C ABI is conservatively maintained — the last major source-incompatible break was the v1.0.0 release in 2015, which renamed the "client connection preface" macros to "client magic" and made the library send the 24-byte magic itself[^6]. Post-1.0 upgrades are generally recompile-clean; the churn is at the `*2` API layer (additive) rather than breaking.

**macOS threading caveat.** The README notes that on macOS the tools may need `--disable-threads` to avoid crashes in `nghttpd`/`nghttpx`/`h2load` — a long-standing platform-specific rough edge.

## When to Use / When Not

**Use when:**
- You need to embed HTTP/2 into an existing C/C++ application or event loop and want full control over transport and TLS.
- You want a standalone HPACK codec.
- You need `h2load` to benchmark an HTTP/2 or HTTP/3 endpoint, or `nghttpx` as a protocol-translating TLS reverse proxy.
- You're a library like curl or a server like Apache httpd that needs a battle-tested framing layer.

**Avoid when:**
- You want an HTTP/2 client with batteries included — use a higher-level library (curl's easy API, or a language-native stack) instead of wiring callbacks yourself.
- You're in a garbage-collected language: use the platform's native HTTP/2 support rather than FFI into libnghttp2.
- You only need an HTTP/2 *server* and don't want to hand-roll one — a full server (nginx, Caddy, H2O) is a better fit than building on the framing library.

## Alternatives

- curl/curl — for an HTTP client, use curl instead of driving libnghttp2 directly; curl embeds nghttp2 and hides the callback machinery.
- h2o/h2o — use when you want a complete high-performance HTTP/2 server rather than a framing library plus your own I/O.
- ngtcp2/nghttp3 — the sibling projects; use these (with nghttp2) when you need QUIC and HTTP/3 rather than HTTP/2 alone.
- litespeedtech/ls-hpack or an in-language HPACK crate — use when you need only header compression and don't want a C dependency.
- golang net/http2 (golang/net) — use when you're in Go and want HTTP/2 without any C FFI.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013-07 | Repository created; forked from `spdylay`[^1]. |
| 1.0.0 | 2015-08 | First stable release; "client magic" API rename, library sends preface[^6]. |
| 1.x | 2016–2021 | HPACK public API, nghttpx maturation, HTTP/3 support added via ngtcp2/nghttp3[^3]. |
| — | 2022 | Codebase tracking RFC 9113 (obsoletes RFC 7540); RFC 7540 priorities deprecated[^2]. |
| 1.69.0 | 2026-04-19 | Latest release at time of writing; modern C++23 toolchain required for tools[^4]. |

## References

[^1]: nghttp2 README — "The nghttp2 code base was forked from the spdylay project." https://github.com/nghttp2/nghttp2
[^2]: RFC 9113, "HTTP/2" (obsoletes RFC 7540), and RFC 7541, "HPACK". https://datatracker.ietf.org/doc/html/rfc9113
[^3]: ngtcp2 (QUIC) and nghttp3 (HTTP/3), sibling projects by the same author. https://github.com/ngtcp2/ngtcp2
[^4]: nghttp2 README build requirements — "C++23 compliant compiler is required. At least g++ >= 14 and clang++ >= 19 are known to work." https://github.com/nghttp2/nghttp2#requirements
[^5]: nghttp2 issue #762 — jemalloc malloc replacement unsupported on Alpine/musl. https://github.com/nghttp2/nghttp2/issues/762
[^6]: nghttp2 README, "Migration from v0.7.15 or earlier". https://github.com/nghttp2/nghttp2#migration-from-v0715-or-earlier

## Tags

c, cpp, http2, http3, quic, hpack, networking, library, proxy, load-testing, tls, protocol
