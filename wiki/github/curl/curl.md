# curl/curl

> The command-line tool and C library (libcurl) that moves data over URLs — installed on billions of devices and roughly every operating system that ships.

[GitHub repo](https://github.com/curl/curl) ·
[Official website](https://curl.se/) ·
[License: curl](https://curl.se/docs/copyright.html)

## Overview

curl is two things sharing a repository: the `curl` command-line client and `libcurl`, the C library that does the actual work. Daniel Stenberg started it in 1996 as `httpget`, renamed it `curl` in 1998, and split the transfer engine out as libcurl around 2000[^1]. He remains the lead maintainer and the project's dominant author nearly three decades on — an unusually long single-hand stewardship for infrastructure this widely deployed.

The defining fact about curl is ubiquity. libcurl is a dependency of git, PHP, most Linux distributions, macOS and Windows (which both bundle `curl.exe`/`curl`), cars, TVs, printers, routers, games, and countless application backends. This is the project's identity and its constraint: because it runs everywhere, it targets old C (C89-era portability), avoids dependencies, and holds a near-absolute commitment to backward compatibility. An API or ABI break in libcurl would ripple through the entire software supply chain, so the project effectively never breaks them[^2].

The tension is scope versus safety. curl speaks over 25 protocols (HTTP through WebSocket, FTP, SMTP, MQTT, TFTP, DICT, and more) and integrates with a dozen swappable TLS and SSH backends. That surface area, written in memory-unsafe C and parsing hostile network input, is why curl has a long CVE history despite careful engineering — the maintainers treat security as a permanent, funded process rather than a solved problem.

## Getting Started

curl ships preinstalled on macOS, Windows 10+, and most Linux distributions. If you need it:

```bash
# Debian/Ubuntu
sudo apt install curl
# macOS (Homebrew, newer than the system copy)
brew install curl
```

Command-line usage:

```bash
# GET and print to stdout
curl https://api.example.com/status

# POST JSON, follow redirects, fail on HTTP errors, show headers
curl -sS --fail-with-body -L \
  -H "Content-Type: application/json" \
  -d '{"name":"curl"}' \
  https://api.example.com/items

# Download to a file, resume if interrupted
curl -C - -o archive.tar.gz https://example.com/archive.tar.gz
```

Minimal libcurl (C), the "easy" interface:

```c
#include <curl/curl.h>

int main(void) {
  CURL *curl = curl_easy_init();
  if (curl) {
    curl_easy_setopt(curl, CURLOPT_URL, "https://example.com");
    curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L);
    CURLcode res = curl_easy_perform(curl);
    if (res != CURLE_OK)
      fprintf(stderr, "%s\n", curl_easy_strerror(res));
    curl_easy_cleanup(curl);
  }
  return 0;
}
```

## Architecture / How It Works

libcurl exposes three C APIs layered over one transfer engine:

1. **easy interface** (`curl_easy_*`) — synchronous, one transfer per call. A `CURL *` handle carries the options set via `curl_easy_setopt`; `curl_easy_perform` blocks until done. Options are the entire configuration model: hundreds of `CURLOPT_*` flags rather than a struct.
2. **multi interface** (`curl_multi_*`) — non-blocking, many transfers over one thread. The application drives the event loop (via `curl_multi_poll`/`_perform` or by feeding curl file descriptors from its own `select`/`epoll`/`libuv` loop). This is how servers do concurrent I/O without a thread per connection.
3. **share interface** (`curl_share_*`) — lets multiple easy handles share cookies, DNS cache, connection pool, and TLS session state.

**Backend abstraction is the core design idea.** curl itself contains no TLS or SSH implementation. At build time you select one (or several) backends and curl talks to them through an internal vtable[^3]:

- TLS: OpenSSL, GnuTLS, wolfSSL, mbedTLS, Schannel (Windows), rustls, BearSSL, and others. Multiple can be compiled in and chosen at runtime.
- SSH (SCP/SFTP): libssh2, libssh, or wolfSSH.
- HTTP/3 + QUIC: ngtcp2, quiche, or OpenSSL's QUIC — still marked experimental in many builds and requires a QUIC-capable TLS backend.

Because backends are pluggable, "curl" behavior can differ between two installs of the same version: the system curl on macOS historically used Secure Transport, on RHEL it uses OpenSSL, on Windows it uses Schannel. TLS behavior, supported ciphers, and CA-store handling follow the backend, not curl.

The connection layer maintains a **reusable connection pool** keyed by host/port/protocol/credentials, with connection reuse, HTTP keep-alive, HTTP/2 multiplexing, and happy-eyeballs (parallel IPv4/IPv6 dialing). Content parsing (chunked encoding, cookies, auth negotiation, redirects) sits above it. The whole thing is C89-portable, single-threaded per handle, and dependency-light by policy.

## Production Notes

**Options are global to a handle and sticky.** Reusing a `CURL *` across transfers is the recommended fast path (it reuses connections and TLS sessions), but options persist — a `CURLOPT_POSTFIELDS` set for one request stays set for the next unless you clear it. Reset with `curl_easy_reset` between logically distinct transfers or accept subtle state leakage.

**The CLI is not a stable scripting API in every respect.** Exit codes and `--write-out` are stable; human-readable stderr text is not. Parse `-w '%{http_code}'`, `--fail`/`--fail-with-body`, and exit status — never scrape progress or error prose. Note `-f/--fail` suppresses the response body on 4xx/5xx; use `--fail-with-body` (curl 7.76+) when you need both non-zero exit and the error payload.

**TLS/CA surprises dominate real-world breakage.** Certificate verification uses the backend's CA store. A statically linked or containerized curl may ship without a CA bundle and fail every HTTPS request with `CURLE_PEER_FAILED_VERIFICATION` until you provide `CURLOPT_CAINFO`/`--cacert`. Do not reach for `-k/--insecure`; it disables verification entirely.

**Security is an ongoing cost, not a footnote.** curl runs a funded bug-bounty program and has published a large number of CVEs over its life. The most publicized recent one, CVE-2023-38545 (a heap buffer overflow in SOCKS5 proxy handling), was fixed in 8.4.0 in October 2023 and forced emergency upgrades across the industry[^4]. Operators should track curl security advisories and keep libcurl patched independently of the OS — the bundled system copy often lags.

**HTTP/3 is real but conditional.** QUIC support depends on which backends were compiled in and is frequently labeled experimental. Do not assume `--http3` works on an arbitrary build; check `curl --version` for the `HTTP3` feature and the listed backends.

**Backward compatibility is a genuine guarantee.** libcurl has held ABI stability for its entire modern life; code written against libcurl years ago still links and runs. Upgrades are low-risk. The cost of that promise is that deprecated options never truly disappear and the option surface only grows.

## When to Use / When Not

**Use when:**
- You need to transfer data over almost any protocol from C/C++ (or via bindings) with one dependency.
- You want a scriptable CLI for HTTP/FTP/etc. that is already installed nearly everywhere.
- You need fine-grained control over TLS, proxies, connection reuse, or streaming that higher-level HTTP libraries hide.
- Portability across obscure or embedded platforms matters.

**Avoid when:**
- You want ergonomic HTTP in a high-level language — use that language's native client (requests, reqwest, axios) unless it wraps libcurl anyway.
- You need a persistent, high-level HTTP/2 or gRPC framework with routing and middleware — curl is a client transport, not a framework.
- You are comfortable with only the CLI's happy path and would rather not manage TLS backend and CA-store variance across platforms.

## Alternatives

- curl/curl vs. **wget / GNU wget** — use wget when you want simple recursive downloading and mirroring with no library ambitions; curl when you need protocols, upload, and a linkable library.
- curl/curl vs. **httpie/httpie** — use HTTPie for a friendlier human CLI for JSON APIs; curl when you need ubiquity, scripts, or a C library.
- curl/curl vs. **openssl s_client** — use s_client to debug a raw TLS handshake; curl for actual data transfer over TLS.
- curl/curl vs. **libwww / neon / libsoup** — narrower, mostly HTTP(S) C libraries; use them only for niche legacy integration — curl outlived them in reach and maintenance.
- curl/curl vs. **hyperium/hyper (Rust)** — use hyper when you want a memory-safe HTTP stack in Rust; note curl can itself use hyper as an HTTP backend.

## History

| Version | Date | Notes |
|---------|------|-------|
| httpget 0.1 | 1996 | First release under its original name[^1]. |
| curl 4.0 | 1998-03 | Renamed to curl. |
| curl 7.1 | 2000-08 | libcurl introduced; library and tool split[^1]. |
| curl 7.16 | 2006–07 | Era of steady protocol/backend expansion (multi interface matured). |
| curl 7.66 | 2019-09 | HTTP/3 (experimental) and parallel transfers in the CLI. |
| curl 8.0.0 | 2023-03-20 | Major-version bump on the 25th anniversary; no ABI break[^5]. |
| curl 8.4.0 | 2023-10-11 | Fixed CVE-2023-38545 (SOCKS5 heap overflow)[^4]. |

## References

[^1]: Daniel Stenberg, "history of curl" — project timeline (httpget 1996 → curl 1998 → libcurl 2000). https://curl.se/docs/history.html
[^2]: Daniel Stenberg, "curl, the C API, and ABI stability." https://curl.se/libcurl/
[^3]: curl documentation, "TLS backends" and the vtls abstraction layer. https://curl.se/docs/ssl-compared.html
[^4]: curl security advisory, CVE-2023-38545 "SOCKS5 heap buffer overflow" — fixed in 8.4.0, 2023-10-11. https://curl.se/docs/CVE-2023-38545.html
[^5]: curl release announcement, "curl 8.0.0" — 2023-03-20. https://curl.se/ch/8.0.0.html

## Tags

c, http, https, cli, library, networking, tls, file-transfer, libcurl, cross-platform, protocol-client
