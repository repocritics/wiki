# h2o/h2o

> An HTTP/1, HTTP/2, and HTTP/3 server written in C, built around latency optimization and usable as an embeddable library.

[GitHub repo](https://github.com/h2o/h2o) ·
[Official website](https://h2o.examp1e.net) ·
[License: MIT](https://github.com/h2o/h2o/blob/master/README.md)

## Overview

H2O is a web server and reverse proxy written in C, started in 2014 by Kazuho Oku at DeNA[^1]. Its design premise is that raw throughput matters less than end-to-end latency, so much of the codebase is about protocol-level features that shorten how long a browser waits: correct HTTP/2 prioritization, cache-aware server push, TLS 1.3 with 0-RTT, and a low-overhead event loop. It ships both as a standalone server (`h2o`) configured with a YAML file and as a C library (`libh2o`) that other programs embed.

The project is the reference home for a cluster of low-level protocol libraries maintained by the same authors: `picotls` (a TLS 1.2/1.3 stack) and `quicly` (a QUIC implementation) live in sibling repositories and are pulled in as submodules to provide H2O's TLS and HTTP/3 support[^2]. This makes H2O less a monolithic server and more the integration point of a protocol-research stack. Lead author Kazuho Oku is an active IETF HTTP/QUIC contributor, and features often appeared in H2O before or alongside their standardization.

The defining tension is maturity versus release discipline. The code is production-proven — Fastly (a copyright holder) deploys H2O-derived software at CDN scale — but the public project has not cut a stable release since **v2.2.6 in August 2019**, and in January 2026 the maintainers pushed an explicit `tag-no-more-releases` tag[^3]. In practice "using H2O" today means building from the `master` branch or trusting a distro package that pins an old tag, not installing a versioned upstream release.

## Getting Started

```bash
# Package managers (may lag master significantly):
brew install h2o                 # macOS
apt-get install h2o              # Debian/Ubuntu

# Or build from source (recommended for HTTP/3 and current fixes):
git clone --recurse-submodules https://github.com/h2o/h2o.git
cd h2o && mkdir build && cd build
cmake -DWITH_MRUBY=on ..
make && sudo make install
```

A minimal `h2o.conf` serving static files and reverse-proxying an API:

```yaml
listen: 8080
listen:
  port: 8443
  ssl:
    certificate-file: /etc/h2o/server.crt
    key-file: /etc/h2o/server.key
    minimum-version: TLSv1.2
hosts:
  "example.com":
    paths:
      "/":
        file.dir: /var/www/html
      "/api":
        proxy.reverse.url: http://127.0.0.1:3000/
access-log: /var/log/h2o/access.log
```

## Architecture / How It Works

H2O is organized as a small core plus a set of **handlers** and **filters** that process each request in a pipeline. The stock handlers are `file` (static files), `proxy` (reverse proxy over HTTP/1 or HTTP/2), `fastcgi`, `redirect`, and `mruby`. The `mruby` handler embeds an mruby (lightweight Ruby) interpreter so request logic — routing, auth, header rewriting — can be scripted without recompiling C[^4]. This is H2O's main extensibility story; there is no dynamic C module ABI in the nginx sense.

Protocol handling is split by version:

- **HTTP/1.x and HTTP/2** are implemented natively in the core. H2O's HTTP/2 implementation is notable for taking prioritization seriously: it implements the HTTP/2 dependency-tree scheduler so that critical resources (CSS, blocking JS) are sent ahead of lower-priority bytes, which is where much of the advertised latency benefit comes from.
- **HTTP/3** is layered on `quicly`, the project's own QUIC stack. It is still labelled experimental in the README and depends on the UDP/QUIC submodule building correctly for your platform.
- **TLS** comes from `picotls`, which provides TLS 1.3 including 0-RTT early data. H2O can also link against OpenSSL/LibreSSL for the TLS 1.2 path.

The standalone server uses its own epoll/kqueue event loop (`h2o evloop`); the library can alternatively be driven by libuv when embedded in a program that already has a libuv loop. **CASPER** (Cache-Aware Server Push) is an H2O-specific feature: the server tracks what a client has likely cached using a Golomb-compressed Bloom filter stored in a cookie, and suppresses HTTP/2 pushes for assets the client already holds. Note that browser vendors have since deprecated HTTP/2 Server Push, so this feature is of diminishing practical value.

## Production Notes

**Release model is the biggest operational caveat.** With no stable release since 2019 and an explicit "no more releases" tag as of 2026, the supported path is building `master` yourself. This means: pin a specific commit, run your own CI against it, and read the git log for behavior changes because there are no versioned changelogs for recent work. Distribution packages (Debian, Homebrew) ship the old v2.2.x line, which predates most of the HTTP/3 and TLS 1.3 work — do not assume a packaged `h2o` has current protocol support.

**HTTP/3 is experimental, not turnkey.** Enabling it requires the `quicly` submodule, UDP socket tuning, and awareness that QUIC's userspace congestion control competes with the kernel for CPU. Treat it as opt-in and load-test it specifically; it is not a drop-in equivalent of the mature HTTP/2 path.

**Configuration surface is smaller than nginx.** H2O covers static serving, reverse proxy, FastCGI, and mruby scripting well, but many nginx conveniences (rich rewrite DSL, a large third-party module ecosystem, mail proxy, extensive stream/L4 features) are absent. Complex request manipulation is expected to move into mruby, which adds an interpreter to the hot path — benchmark it.

**Observability and ecosystem are thin.** Community size, StackOverflow coverage, and third-party tutorials are far smaller than nginx/Caddy. Expect to read source and the official docs rather than find a blog post for your exact problem. The upside is a small, auditable C codebase that has been through Coverity Scan and OSS-Fuzz.

**Reverse-proxy realities.** The proxy handler is solid for HTTP upstreams but is not a full service-mesh L7 proxy — no dynamic upstream discovery, circuit breaking, or gRPC-aware routing of the kind Envoy provides. Use it as a front-end/edge server, not as an internal mesh data plane.

## When to Use / When Not

**Use when:**
- You want best-effort HTTP/2 prioritization and TLS 1.3 / 0-RTT at the edge and are comfortable building from source.
- You need to embed an HTTP server in a C/C++ program (`libh2o`) rather than run a separate process.
- You want a small, auditable, fuzz-tested C codebase and can script request logic in mruby.
- You are experimenting with HTTP/3/QUIC and want a server that tracks IETF work closely.

**Avoid when:**
- You need a vendor-supported, versioned-release cadence with security-patch backports — the no-releases posture is disqualifying for many compliance regimes.
- You rely on a large module ecosystem or extensive community Q&A (nginx wins decisively).
- You need L4/L7 service-mesh features, dynamic upstreams, or gRPC routing (use Envoy).
- You want automatic HTTPS and zero-config TLS issuance (use Caddy).

## Alternatives

- nginx/nginx — the incumbent C server/proxy; vastly larger ecosystem, regular releases, HTTP/3 in current versions. Use instead when you want maturity, modules, and support over protocol-research features.
- caddyserver/caddy — Go server with automatic HTTPS via ACME and simple config. Use instead when TLS automation and operability matter more than raw C performance.
- envoyproxy/envoy — L7 proxy built for service meshes. Use instead when you need dynamic upstreams, gRPC, and mesh data-plane features.
- apache/httpd — traditional, module-rich server. Use instead when you need `.htaccess`, mod_php, or an established enterprise footprint.
- litespeedtech/openlitespeed — event-driven server with nginx-like config. Use instead when you want LiteSpeed compatibility and a maintained release line.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2014-08 | Project started at DeNA by Kazuho Oku[^1]. |
| 2.0.0 | 2016-06-01 | HTTP/2 maturity, TLS 1.3 draft work via picotls. |
| 2.1.0 | 2017-01-18 | Feature release on the 2.x line. |
| 2.2.0 | 2017-04-05 | Start of the long-lived 2.2 stable series. |
| 2.2.6 | 2019-08-13 | Last stable tagged release[^3]. |
| 2.3.0-beta2 | 2019-08-13 | Last beta; HTTP/3/quicly work continued on master. |
| tag-no-more-releases | 2026-01-19 | Maintainers signal the end of tagged releases; master-only from here[^3]. |

## References

[^1]: H2O README and copyright — DeNA Co., Ltd., Kazuho Oku, Fastly, Inc., et al. https://github.com/h2o/h2o/blob/master/README.md
[^2]: quicly (QUIC) and picotls (TLS 1.2/1.3) sibling repositories used as submodules. https://github.com/h2o/quicly · https://github.com/h2o/picotls
[^3]: H2O release/tag list, including `v2.2.6` (2019-08-13) and `tag-no-more-releases` (2026-01-19). https://github.com/h2o/h2o/tags
[^4]: H2O configuration and mruby handler documentation. https://h2o.examp1e.net/configure/mruby.html

## Tags

http-server, reverse-proxy, http2, http3, quic, tls, c, web-server, performance, latency, mruby, edge
