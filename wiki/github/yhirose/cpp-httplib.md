# yhirose/cpp-httplib

> A single-file, header-only C++11 HTTP/HTTPS server and client — blocking I/O, HTTP/1.1 only, drop it in and go.

[GitHub repo](https://github.com/yhirose/cpp-httplib) ·
[Official website](https://yhirose.github.io/cpp-httplib/) ·
[License: MIT](https://github.com/yhirose/cpp-httplib/blob/master/LICENSE)

## Overview

cpp-httplib is a C++11 HTTP/HTTPS library authored and maintained largely by Yuji Hirose. Its entire implementation lives in one header, `httplib.h`: you copy that file into your tree, `#include` it, and you have both a server and a client with no build-system integration, no submodules, and no linked artifacts beyond an optional TLS backend[^1]. That single-file distribution is the whole value proposition — it is the fastest way to add an HTTP endpoint or a REST client to a C++ project without adopting Boost, vcpkg, or a package manager.

The library is deliberately narrow. It implements **HTTP/1.1 only** — there is no HTTP/2 or HTTP/3 — and it uses **blocking socket I/O** with a thread-per-connection model[^2]. The README states both constraints up front, unusually honestly: if you need non-blocking/event-driven I/O for tens of thousands of concurrent connections, the maintainer tells you outright to look elsewhere. cpp-httplib is a tool for internal services, embedded control planes, test servers, tooling endpoints, and moderate-traffic APIs — not a front-line edge server.

The defining tension is convenience versus scale. You trade the C10k concurrency ceiling and a large translation-unit cost (the header is enormous) for zero-friction integration and a small, readable, synchronous programming model. For a great many real workloads that is the right trade; for high-fanout public-facing traffic it is not.

## Getting Started

There is nothing to install — vendor the header:

```bash
curl -O https://raw.githubusercontent.com/yhirose/cpp-httplib/master/httplib.h
```

```cpp
#include "httplib.h"

int main() {
  httplib::Server svr;

  svr.Get("/hi", [](const httplib::Request&, httplib::Response& res) {
    res.set_content("Hello World!", "text/plain");
  });

  // Regex captures and typed path params both work
  svr.Get("/users/:id", [](const httplib::Request& req, httplib::Response& res) {
    res.set_content(req.path_params.at("id"), "text/plain");
  });

  svr.listen("0.0.0.0", 8080);
}
```

Client side is symmetric:

```cpp
httplib::Client cli("http://localhost:8080");
if (auto res = cli.Get("/hi")) {
  // res->status, res->body
}
```

TLS requires a backend define plus linking `libssl`/`libcrypto`:

```cpp
#define CPPHTTPLIB_OPENSSL_SUPPORT   // needs OpenSSL 3.0+
#include "httplib.h"
httplib::SSLServer svr("./cert.pem", "./key.pem");
```

## Architecture / How It Works

The server accepts connections on a listening socket and dispatches each to a worker thread drawn from a fixed thread pool (sized from hardware concurrency by default, overridable via `CPPHTTPLIB_THREAD_POOL_COUNT` or a custom `TaskQueue`). Each worker owns the socket for the full lifetime of the connection and does all read/parse/route/write work synchronously on that thread. Keep-alive connections therefore **hold a pool thread for as long as the connection stays open**. This is the model's central design fact and its central footgun (see Production Notes).

Request handling runs through an ordered handler chain, not a flat router: `pre_routing_handler` → static file serving → route matching → `pre_request_handler` (runs after the route matches but *before* the body is read, so you can reject on auth without buffering a large upload) → the route handler → `post_routing_handler`, with `exception_handler` and `error_handler` on the failure paths[^3]. Routes are matched by exact path, `:name` path params, or ECMAScript regex captures exposed via `req.matches`.

TLS is an abstraction over multiple backends selected at compile time: **OpenSSL** (3.0+), **Mbed TLS** (2.x/3.x), and **wolfSSL** (5.x, built with `--enable-opensslall`); **BoringSSL** compiles under the OpenSSL define on a best-effort basis with known differences (its headers require C++14, and it does SAN-only hostname verification with no CN fallback)[^1]. On macOS and Windows the client integrates with the OS certificate store automatically (Keychain via `CoreFoundation`/`Security`; CryptoAPI chain verification on Windows), each toggleable off with a compile define.

Beyond plain request/response, the library ships a Stream API for chunked/streamed bodies, Server-Sent Events, and WebSocket support, all in the same header. There is no external dependency for the non-TLS core — it is straight POSIX/Winsock sockets plus the C++11 standard library.

## Production Notes

**Thread-pool exhaustion is the real risk.** Because every open connection pins a worker thread, a modest number of slow or idle keep-alive clients can consume the entire pool and starve new requests — a Slowloris-style denial of service is achievable without much effort. Put cpp-httplib behind a reverse proxy (nginx, Envoy) that terminates untrusted connections, buffers slow clients, and enforces timeouts; tune `CPPHTTPLIB_THREAD_POOL_COUNT`, `set_keep_alive_max_count`, and the read/write timeouts to your concurrency budget. Do not expose it raw to the open internet for high-fanout traffic.

**Compile-time cost is significant.** `httplib.h` is a very large header; `#include`-ing it in many translation units inflates build times noticeably. The repo provides a `split.py` script that separates the header into declarations plus a single compiled `httplib.cc`, which keeps the heavy implementation out of every TU. Use it once your project includes the header in more than a couple of places.

**Static file serving has sharp edges.** The mount-point methods (`set_mount_point`, etc.) are explicitly **not thread-safe** — configure mounts before `listen()`, not from request handlers. On POSIX the server does reject symlink escapes outside the mounted base, but directory permissions remain the application's responsibility.

**Exception handling can crash the server.** If you install a custom `exception_handler`, you must include a `catch (...)` fallback for the rethrown `std::exception_ptr`; an uncaught exception propagating out of a handler will terminate the process. The README flags this explicitly.

**TLS operational caveats.** SIGPIPE cannot be fully suppressed for OpenSSL's internal I/O on all platforms — install your own SIGPIPE handler if process termination on broken pipe is unacceptable. With Mbed TLS/wolfSSL, `get_ca_certs()`/`get_ca_names()` only reflect CAs loaded via `load_ca_cert_store()`, not those from `set_ca_cert_path()` or the system store.

**32-bit is unsupported and unaudited.** The maintainer states 32-bit builds receive no security review and that security reports affecting only 32-bit platforms are closed without action. Target 64-bit only.

## When to Use / When Not

**Use when:**
- You want an HTTP server or client in a C++ project with zero build integration.
- Traffic is internal, moderate, or proxied — tooling endpoints, control planes, embedded devices, test doubles, microservices behind a load balancer.
- You value a small, synchronous, readable programming model over maximum throughput.
- You need HTTPS with a choice of TLS backend but not HTTP/2.

**Avoid when:**
- You need non-blocking/async I/O or C10k+ concurrency on a single process.
- You need HTTP/2 or HTTP/3, gRPC, or multiplexed streams.
- You're terminating untrusted public traffic directly with no proxy in front.
- Build times matter and you can't afford a very large header across many translation units (mitigable with `split.py`, but consider a compiled library).

## Alternatives

- boostorg/beast — use instead when you need non-blocking async I/O built on Boost.Asio and are already in the Boost ecosystem.
- drogon/drogon — use instead when you need a high-concurrency async framework (event loop, ORM, HTTP/2) for C10k+ workloads.
- oatpp/oatpp — use instead when you want a structured async web framework with API/Swagger generation and DI.
- uNetworking/uWebSockets — use instead when you need very large numbers of concurrent WebSocket/HTTP connections per core.
- pocoproject/poco — use instead when you want a broad batteries-included C++ platform library, not just an HTTP layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2012-09-22 | Repository created[^4]. |
| 0.x | ongoing | Continuous `v0.x` tagging; no `1.0` — API evolves within the 0.x line without a formal major-version break. |
| — | 2026-07-12 | Actively maintained; last push per GitHub API[^4]. WebSocket, SSE, Stream API, and multi-backend TLS all present in the current header. |

## References

[^1]: cpp-httplib README — single-file header-only distribution, main features, and TLS backend matrix (OpenSSL/Mbed TLS/wolfSSL/BoringSSL). https://github.com/yhirose/cpp-httplib
[^2]: cpp-httplib README, "IMPORTANT" notice — blocking socket I/O and HTTP/1.1-only scope. https://github.com/yhirose/cpp-httplib
[^3]: cpp-httplib README, "Handler execution order" — the pre-routing / pre-request / route / post-routing chain and exception/error handlers. https://github.com/yhirose/cpp-httplib
[^4]: GitHub REST API, `repos/yhirose/cpp-httplib` — created 2012-09-22, last push 2026-07-12, MIT license, ~16.7k stars (fetched 2026-07-15). https://api.github.com/repos/yhirose/cpp-httplib

## Tags

cpp, cpp11, http, https, header-only, http-server, http-client, tls, websocket, server-sent-events, networking, single-file
