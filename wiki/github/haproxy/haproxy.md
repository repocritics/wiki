# haproxy/haproxy

> An event-driven, single-purpose TCP/HTTP load balancer and reverse proxy, tuned for extreme connection counts and low latency.

[GitHub repo](https://github.com/haproxy/haproxy) ·
[Official website](https://www.haproxy.org) ·
[License: GPL-2.0-or-later](https://github.com/haproxy/haproxy/blob/master/LICENSE)

## Overview

HAProxy (High Availability Proxy) is a load balancer and reverse proxy for TCP
and HTTP applications, written in C and maintained by Willy Tarreau since the
early 2000s[^1]. It does one thing — move bytes between clients and backend
servers as fast as the kernel allows — and has resisted scope creep for two
decades. It is the load balancer behind a large share of high-traffic sites and
CDNs, and the reference against which "how many requests per second" claims are
usually measured.

The GitHub repository is a **mirror** of the canonical tree at
`git.haproxy.org`; its star count (thousands) badly understates the project's
reach, because HAProxy predates GitHub by roughly fifteen years and its
development happens on a mailing list, not in pull requests. Do not read the
GitHub activity as the project's activity — read the ML and the release
cadence[^2].

The defining tradeoff is configuration model. HAProxy is driven by a single
static `haproxy.cfg` file plus a runtime API; it has no dynamic control plane,
no service-discovery layer, and no plugin marketplace. In exchange you get
predictable behavior, tiny memory footprint, and latency that stays flat under
load. Teams that want config-as-API and mesh integration reach for Envoy or
Traefik; teams that want a fast, boring, auditable data plane stay on HAProxy.

## Getting Started

```bash
# Debian / Ubuntu (packaged; often lags upstream by a release or two)
apt-get install haproxy

# From source — pick a TARGET and the features you need
make -j$(nproc) TARGET=linux-glibc USE_OPENSSL=1 USE_PCRE2=1 USE_LUA=1
make install
```

```haproxy
# /etc/haproxy/haproxy.cfg — L7 HTTP load balancing with health checks
global
    maxconn 20000
    log stdout format raw local0

defaults
    mode    http
    log     global
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend web
    bind :80
    bind :443 ssl crt /etc/haproxy/site.pem
    default_backend app

backend app
    balance roundrobin
    option httpchk GET /healthz
    server s1 10.0.0.1:8080 check
    server s2 10.0.0.2:8080 check
```

Validate before reloading: `haproxy -c -f /etc/haproxy/haproxy.cfg`.

## Architecture / How It Works

HAProxy is a **single-process, event-driven** engine built on `epoll` (Linux),
`kqueue` (BSD/macOS), or `event ports` (Solaris/illumos). It was single-threaded
for most of its life; native multithreading landed in **1.8 (2017)** via the
`nbthread` directive, and modern deployments pin threads to cores with
`cpu-map`[^3]. There is no per-connection thread or process — one worker handles
tens of thousands of concurrent connections in a tight I/O loop.

Config is organized into `global`, `defaults`, and `frontend` / `backend` /
`listen` sections. Request routing is expressed with **ACLs** (boolean match
expressions over any request attribute), backend selection, `map` files, and
**stick-tables** — in-memory key/value stores used for session persistence,
rate limiting, and connection tracking. Stick-tables can be synchronized across
HAProxy instances with the **peers protocol**[^4].

Internally, HTTP is handled through **HTX**, a native structured HTTP
representation introduced in 1.9 and made the default in **2.0 (2019)**. HTX
decoupled the parser from the wire format and is what makes end-to-end HTTP/2
and, later, HTTP/3 possible without translating everything back to HTTP/1
text[^5]. **QUIC / HTTP/3** support was added experimentally around 2.6 and has
matured since; HAProxy ships its own QUIC stack rather than depending on a third
party, but it requires a TLS library that exposes the QUIC API (quictls or
compatible), which is the single most common build-time surprise.

A **master-worker** model supervises workers and enables seamless reloads: on
reload the master starts new workers and drains the old ones, transferring
listening sockets so in-flight connections are not dropped. The **runtime API**
(a stats/admin socket) lets you enable, disable, and reconfigure servers, adjust
weights, and inspect stick-tables live, without a reload at all.

## Production Notes

- **Reloads fork, they don't hot-swap.** Every `reload` spawns fresh workers and
  keeps old ones alive until their connections drain. Long-lived connections
  (WebSockets, gRPC streams) can pin old workers for a long time; a busy site
  reloading frequently can accumulate processes and memory. Use `hard-stop-after`
  to bound the drain window.
- **Config is static by design.** There is no built-in dynamic backend discovery.
  For elastic backends you use `server-template` + DNS SRV resolution, the
  runtime API, or the separate **Data Plane API** — none of which are as
  turnkey as Envoy's xDS or Traefik's provider model.
- **File-descriptor and connection limits matter.** `maxconn`, the process
  `ulimit -n`, and kernel `somaxconn` must all be raised together; the effective
  ceiling is the smallest of them, and mis-sizing shows up as silent connection
  refusal under load.
- **Stick-tables are sized at config time.** Their memory is pre-allocated from
  `size`; oversizing wastes RAM, undersizing evicts entries and breaks
  persistence/rate-limiting silently.
- **Logging defaults to syslog.** Historically HAProxy logs over UDP syslog,
  which drops under pressure and confuses operators expecting stdout. Modern
  versions can log to stdout/ring buffers — configure it explicitly.
- **Community vs Enterprise.** This repo is the community edition. HAProxy
  Technologies sells HAProxy Enterprise with extra modules, a bundled WAF, and
  support; some blog features and timelines refer to the Enterprise build, not
  this tree — verify a feature is in the open source version before relying on it.
- **Version and support model.** Roughly two releases per year. Certain releases
  are designated long-term-supported and maintained for years (2.2, 2.4, 2.6,
  2.8, 3.0 among them); consult `BRANCHES` in-repo before pinning a version[^2].

## When to Use / When Not

**Use when:**
- You need L4 or L7 load balancing / TLS termination at high request rates with
  flat, predictable latency.
- You want fine-grained routing (ACLs), rate limiting, and DDoS mitigation with
  stick-tables, in a single auditable config file.
- You value a small, dependency-light, long-lived data plane over a dynamic
  control plane.

**Avoid when:**
- You need config-as-API, service mesh, or automatic service discovery as a
  first-class feature — Envoy or Traefik fit better.
- Your primary need is a web server, static file serving, or an HTTP cache —
  nginx or Varnish are more natural.
- You want to extend the proxy in a high-level language with a large plugin
  ecosystem (Lua scripting exists, but it is not Envoy's filter model).

## Alternatives

- envoyproxy/envoy — use when you need a dynamic xDS control plane, a service
  mesh data plane, or programmatic reconfiguration without reloads.
- nginx/nginx — use when you also need a web server, static content, or HTTP
  caching in the same process, not just load balancing.
- traefik/traefik — use when you want Kubernetes-native, auto-discovering dynamic
  config out of the box and can accept higher overhead.
- caddy — use when automatic HTTPS and a minimal config are the priority over raw
  throughput.
- Varnish (varnishcache/varnish-cache) — use when the core requirement is a
  high-performance HTTP cache rather than balancing.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | ~2001–2010 | Original single-threaded event-driven proxy by Willy Tarreau[^1]. |
| 1.5 | 2014-06 | Native SSL/TLS, IPv6, PROXY protocol, HTTP keep-alive. |
| 1.8 | 2017-11 | Multithreading, HTTP/2, seamless reloads, runtime server management[^3]. |
| 2.0 | 2019-06 | HTX default, Layer 7 retries, Prometheus exporter, Data Plane API[^5]. |
| 2.4 | 2021-05 | Long-term-supported; HTTP/2 improvements, `server-state` refinements. |
| 2.6 | 2022-05 | Experimental QUIC / HTTP/3 support introduced. |
| 2.8 | 2023-05 | Long-term-supported release; QUIC and performance hardening. |
| 3.0 | 2024-05 | Long-term-supported; QUIC maturation, config and reload improvements[^2]. |

## References

[^1]: HAProxy project — history and authorship, Willy Tarreau. https://www.haproxy.org/
[^2]: HAProxy canonical git and release branches (`git.haproxy.org`, in-repo `BRANCHES`). https://git.haproxy.org/
[^3]: HAProxy 1.8.0 release announcement (multithreading, HTTP/2, seamless reloads) — 2017-11. https://www.mail-archive.com/haproxy@formilux.org/
[^4]: HAProxy configuration manual — ACLs, stick-tables, and peers. http://docs.haproxy.org/
[^5]: HAProxy 2.0 release notes (HTX default, Data Plane API) — 2019-06. https://www.haproxy.com/blog/haproxy-2-0-and-beyond/

## Tags

c, load-balancer, reverse-proxy, http, tls, high-availability, networking, proxy, http2, quic, infrastructure
