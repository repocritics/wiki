# envoyproxy/envoy

> An L3/L4 and L7 proxy written in C++, designed to be the programmable data plane of a service mesh — configured at runtime over a streaming gRPC API rather than a config file.

[GitHub repo](https://github.com/envoyproxy/envoy) ·
[Official website](https://www.envoyproxy.io) ·
[License: Apache-2.0](https://github.com/envoyproxy/envoy/blob/main/LICENSE)

## Overview

Envoy is a network proxy originally built at Lyft by Matt Klein and open-sourced in September 2017, then donated to the Cloud Native Computing Foundation, where it became a graduated project in 2018[^1]. It occupies the same slot as nginx or HAProxy — a reverse proxy and load balancer — but was designed from the start for a world of many small services rather than a handful of web servers. Its defining idea is that configuration (which upstreams exist, how to route to them, what filters to run) is not a file you reload, but state pushed to the proxy at runtime over the xDS APIs.

That design is why Envoy became the near-universal data plane for service meshes: it is the sidecar in Istio and the engine under Contour, Emissary-ingress (formerly Ambassador), Gloo, and the CNCF's own Envoy Gateway[^2]. You rarely deploy raw Envoy for a small app — you deploy something that generates Envoy config for you.

The tension: Envoy is enormously capable and correspondingly heavy. Its configuration surface is one of the largest in infrastructure software, its YAML is verbose to the point of being unwritable by hand for non-trivial cases, and getting dynamic config means running a control plane (Istio, go-control-plane, or your own). For an edge proxy in front of one service, this is overkill; for a fleet of hundreds of services needing consistent observability, retries, mTLS, and traffic shaping, it is the reference implementation.

## Getting Started

```bash
# Run the official image with a static bootstrap config
docker run --rm -d -p 10000:10000 -p 9901:9901 \
  -v "$PWD/envoy.yaml:/etc/envoy/envoy.yaml" \
  envoyproxy/envoy:v1.34-latest
```

```yaml
# envoy.yaml — minimal static config: listen on 10000, proxy to example.com
static_resources:
  listeners:
  - address: { socket_address: { address: 0.0.0.0, port_value: 10000 } }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { host_rewrite_literal: example.com, cluster: svc }
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
  - name: svc
    type: STRICT_DNS
    load_assignment:
      cluster_name: svc
      endpoints:
      - lb_endpoints:
        - endpoint: { address: { socket_address: { address: example.com, port_value: 80 } } }
```

The admin interface on `:9901` (`/stats`, `/config_dump`, `/clusters`) is the single most useful thing to learn early.

## Architecture / How It Works

**Threading model.** Envoy runs a single process with one main thread plus N worker threads (typically one per core). Each worker owns a slice of connections and runs an event loop (libevent) that is almost entirely non-blocking. State that would otherwise need locking — cluster membership, stats, config — is kept in thread-local storage and updated by posting to each worker, so the hot path is effectively lock-free[^3]. This is why Envoy scales predictably under load but also why a single slow blocking call in a filter can stall a whole worker.

**xDS / the universal data plane API.** The control surface is a family of gRPC (or REST) discovery services: LDS (listeners), RDS (routes), CDS (clusters), EDS (endpoints), SDS (secrets), plus aggregated ADS for ordering guarantees[^4]. A control plane streams updates and Envoy applies them without a restart or dropped connection. The API moved from v2 to v3 (v2 removed in 2021); the proto definitions in `api/` are the actual contract that Istio and every other control plane target.

**Filter chains.** Requests pass through network filters (L3/L4) and, for HTTP, the HTTP Connection Manager plus a chain of HTTP filters (routing, auth, rate limiting, fault injection, the terminal router filter). Extensibility is via compiled C++ extensions, Lua, or proxy-wasm (WASM) modules — the last lets you ship filters without recompiling Envoy, at a runtime cost.

**Hot restart.** Envoy can start a new process that inherits the listening sockets and drains the old one, allowing binary/config upgrades with zero dropped connections[^5] — a feature predating Kubernetes-style rolling deploys and less relevant when the orchestrator handles it.

The build is Bazel-based and large; compiling Envoy from source is a multi-hour, high-memory affair, which is why almost everyone consumes the official images.

## Production Notes

- **You need a control plane for dynamic config.** Raw Envoy with static YAML works, but the value proposition (runtime endpoint discovery, mTLS rotation, traffic shifting) requires Istio, go-control-plane, or equivalent. Budget for operating that, not just Envoy.
- **Stats cardinality is a real cost.** Envoy emits per-cluster, per-endpoint, per-route metrics. On a large mesh this explodes memory and overwhelms Prometheus; `stats_matcher` / tag extraction to drop unused stats is standard hardening, not an optimization.
- **Memory scales with config and connections.** Each cluster and listener has fixed overhead, and buffering (retries buffer request bodies) adds per-request memory. Sidecars in dense pods are frequently the memory-limited component.
- **The learning curve is steep and the errors are cryptic.** `config_dump` and the admin `/clusters` and `/stats` endpoints are how you debug; NACKs from a bad xDS push are silent unless you watch the control plane's stats.
- **Version cadence is quarterly** and each release has a defined support window (roughly the current plus prior few minor versions)[^6]. Deprecations are aggressive; running far behind means fighting removed config fields on upgrade.
- **TLS termination is CPU-bound.** BoringSSL under the hood is fast, but mTLS everywhere in a mesh is a measurable tax; size for it.

## When to Use / When Not

**Use when:**
- You run a service mesh or many services needing consistent L7 routing, retries, circuit breaking, outlier detection, and observability.
- You need config to change at runtime (endpoint churn, canary shifts, secret rotation) without reloading.
- You want deep HTTP/2, HTTP/3 (QUIC), and gRPC support with rich, uniform telemetry.

**Avoid when:**
- You have one or a few services and just need a reverse proxy — nginx, HAProxy, or Caddy are dramatically simpler.
- You cannot invest in a control plane or the operational learning curve.
- You are memory-constrained per instance and don't need dynamic config or L7 policy.

## Alternatives

- nginx/nginx — mature, lightweight reverse proxy; use instead when you want a static, well-understood edge proxy without dynamic xDS or mesh ambitions.
- haproxy/haproxy — top-tier L4/L7 load balancer with lower resource use; use when raw throughput and simplicity matter more than runtime programmability.
- traefik/traefik — Go proxy with easy service discovery and auto-TLS; use for Kubernetes/Docker ingress where config-by-labels beats Envoy's verbosity.
- cloudflare/pingora — Rust proxy framework; use when you're building a custom proxy in code rather than configuring a general-purpose one.
- linkerd/linkerd2-proxy — purpose-built Rust mesh sidecar; use when you want a lighter service mesh and reject Envoy/Istio's complexity.

## History

| Version | Date | Notes |
|---------|------|-------|
| Open source | 2017-09 | Released by Lyft, joins CNCF[^1]. |
| Graduated | 2018-11 | Third CNCF project to graduate, after Kubernetes and Prometheus. |
| API v3 | 2020 | xDS v3 stabilizes; v2 API removed in 2021[^4]. |
| HTTP/3 | 2022 | QUIC / HTTP/3 support reaches production readiness. |
| — | quarterly | Minor releases on a ~3-month cadence with defined support windows[^6]. |

## References

[^1]: CNCF, "CNCF Hosts Envoy" — 2017-09-13. https://www.cncf.io/blog/2017/09/13/cncf-hosts-envoy/
[^2]: Envoy Gateway project. https://gateway.envoyproxy.io/
[^3]: Matt Klein, "Envoy threading model." https://medium.com/@mattklein123/envoy-threading-model-a8d44b922310
[^4]: Envoy docs, "xDS REST and gRPC protocol." https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol
[^5]: Matt Klein, "Envoy hot restart." https://medium.com/@mattklein123/envoy-hot-restart-1d16b14555b5
[^6]: Envoy release process and support. https://github.com/envoyproxy/envoy/blob/main/RELEASES.md

## Tags

c-plus-plus, proxy, load-balancer, service-mesh, cncf, networking, http2, grpc, sidecar, observability, cloud-native, xds
