# TykTechnologies/tyk

> A Go API gateway and reverse proxy — auth, rate limiting, quotas, and analytics in one batteries-included binary, backed by Redis.

[GitHub repo](https://github.com/TykTechnologies/tyk) ·
[Official website](https://tyk.io) ·
[License: MPL-2.0](https://github.com/TykTechnologies/tyk/blob/master/LICENSE) (dual-licensed)

## Overview

Tyk Gateway is an API gateway written in Go, open-sourced in 2014 by Tyk Technologies (founder Martin Buhr)[^1]. It sits in front of upstream services as a reverse proxy and layers on the things every API program eventually needs: authentication (keys, JWT, OIDC, mTLS, HMAC, OAuth2), rate limiting, quotas, request/response transformation, versioning, and per-consumer access control. The open-source gateway is fully functional on its own — Tyk advertises "no feature lockout" — and is the same binary that underpins the company's paid Self-Managed and Cloud products.

The defining tension in Tyk is **open-source gateway vs. commercial control plane**. The gateway is genuinely open (MPL-2.0), but the management surface most teams expect — a GUI dashboard, developer portal, multi-data-center bridge, RBAC — is closed commercial software. Running Tyk purely open-source means configuring APIs through JSON files or the Gateway REST API, wiring your own CI, and reading raw analytics out of Redis yourself. Teams that discover this after adopting Tyk often feel the OSS/commercial line was drawn exactly where operational convenience begins.

The second recurring theme is the **Redis hard dependency**. Redis is not an optional analytics sink; it is on the hot path for tokens, rate-limit counters, quotas, and the analytics buffer. A Tyk deployment's availability is bounded by its Redis availability.

## Getting Started

The maintainers recommend the Docker Compose bundle (gateway + Redis) as the fastest path:

```bash
git clone https://github.com/TykTechnologies/tyk-gateway-docker
cd tyk-gateway-docker
docker-compose up
```

Verify the gateway is live:

```bash
curl localhost:8080/hello
# {"status":"pass","version":"v5.x","description":"Tyk GW"}
```

A minimal Tyk OAS API definition proxies an upstream and enables auth. With the newer OpenAPI-native format, gateway config lives under an `x-tyk-api-gateway` extension inside a standard OAS document:

```json
{
  "openapi": "3.0.3",
  "info": { "title": "httpbin", "version": "1.0.0" },
  "x-tyk-api-gateway": {
    "info": { "name": "httpbin", "state": { "active": true } },
    "upstream": { "url": "https://httpbin.org/" },
    "server": { "listenPath": { "value": "/httpbin/", "strip": true } }
  }
}
```

POST it to the Gateway API (or drop it in `apps/` in file-based mode) and hot-reload with `/tyk/reload/group`.

## Architecture / How It Works

At the core, Tyk is Go's `net/http/httputil.ReverseProxy` wrapped in a configurable **middleware chain**. Each incoming request walks an ordered pipeline — version check, auth, rate limit, quota, allow/block/ignore lists, transforms, then the proxy hop — and every stage can short-circuit. The chain is what you customize.

Key internals:

- **Storage layer is Redis.** API keys, OAuth clients, rate-limit and quota counters, and buffered analytics records all live in Redis. Configuration hot-reloads propagate across a gateway cluster via Redis pub/sub. This makes Redis both the coordination bus and the session store.
- **Two API definition formats coexist.** The original **Tyk Classic** format is a bespoke JSON schema. Since Tyk 4.1 there is **Tyk OAS**, where the OpenAPI document is the source of truth and Tyk config is an extension inside it. Both are supported; docs, tooling, and features are split across the two, and not every Classic feature has an OAS equivalent yet.
- **Plugin/middleware extensibility** comes in several flavors: a built-in JS virtual machine (goja/otto) for lightweight middleware, and **coprocess plugins** over gRPC that let you write middleware in any language. Native **Go plugins** (`.so` via `plugin` package) are also supported for maximum performance.
- **Rate limiting** offers multiple algorithms including a Redis-backed distributed sliding-window limiter and a lower-overhead in-memory option; the distributed limiter trades a Redis round-trip for accuracy across a cluster.
- **Dual licensing in-tree.** Everything outside the `ee/` directory is MPL-2.0; code under `ee/` is under a commercial license[^2]. The repository builds as one binary, but enabling `ee/` features requires a license.

The companion OSS projects — Tyk Pump (analytics purger), Tyk Operator (Kubernetes CRDs), Tyk Identity Broker, Tyk Sync — are separate repos that together form the open-source operational story the gateway alone doesn't provide.

## Production Notes

**Redis is a first-class dependency, plan it like one.** Rate limits, quotas, and token lookups hit Redis synchronously. A Redis stall becomes gateway latency; a Redis outage breaks auth and limiting. Run Redis in HA (Sentinel or cluster), size it for your key cardinality, and monitor it as a tier-1 service. The analytics buffer also accumulates in Redis until Tyk Pump drains it — if Pump falls behind, Redis memory grows.

**Go plugins are a known footgun.** Native `.so` plugins must be compiled with the *exact* same Go version, build tags, and dependency versions as the gateway binary, and must be recompiled on every gateway upgrade — a mismatch fails to load at runtime. Most teams that need custom middleware use **gRPC coprocess plugins** instead, trading a network hop for build-independence and language freedom.

**JSVM plugins are limited.** The embedded JS engine is not Node — no npm ecosystem, no full standard library, and each VM is single-threaded. Fine for header tweaks and small logic, wrong for anything heavy.

**The OSS management gap is real.** Without the commercial Dashboard there is no official GUI; you manage APIs via JSON files or the Gateway REST API, and analytics come out of Redis as raw records only (aggregation is Pump + a backend you supply, or the paid Dashboard). Budget for GitOps tooling (Tyk Sync / Operator) up front rather than discovering the gap later.

**Two config formats create drift.** Mixing Tyk Classic and Tyk OAS APIs in one deployment means two mental models and two documentation paths. Pick OAS for new work if your features are supported; migrating existing Classic definitions is a manual, feature-by-feature exercise.

**Building from source** currently requires Go 1.22 for `master`, with official support for `Linux/amd64`, `Linux/i386`, and `Linux/arm64`; tests require a local Redis on the default port.

## When to Use / When Not

**Use when:**
- You want auth, rate limiting, quotas, and analytics as one self-hostable Go binary without stitching them together.
- You need multi-protocol support (REST, GraphQL, gRPC, TCP) behind one gateway.
- You run on Kubernetes and want declarative API management via the Tyk Operator CRDs.
- You are comfortable managing config as JSON/GitOps and running Redis as core infrastructure.

**Avoid when:**
- You want a polished management GUI, developer portal, or RBAC out of the box without paying — those are commercial.
- You cannot run or don't want to operate Redis as a highly-available dependency.
- You need heavy custom middleware but can't accept the Go-plugin build coupling or the gRPC-plugin network hop.
- You want a stateless, database-free gateway configured entirely from a static file at boot.

## Alternatives

- kong/kong — OpenResty/Lua on nginx with a large plugin hub; use instead when you want the biggest off-the-shelf plugin ecosystem and an nginx foundation.
- apache/apisix — nginx + etcd, dynamic config, strong raw throughput; use when you prefer etcd-driven config and hot reconfiguration without Redis.
- envoyproxy/envoy — an L7 proxy and service-mesh data plane, not a batteries-included gateway; use when you need a programmable proxy substrate rather than turnkey API management.
- luraproject/lura (KrakenD) — stateless, declarative, no datastore; use when you want a config-file-only aggregation gateway with no Redis and no per-request state.
- emissary-ingress/emissary — Envoy-based Kubernetes-native API gateway; use when you want an Envoy control plane wired to Kubernetes ingress.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-05 | Repository opened; Tyk open-sourced[^1]. |
| 2.0 | 2016 | Dashboard, policies, broader plugin support. |
| 3.0 | 2020 | GraphQL support, plugin and analytics improvements. |
| 4.0 | 2022 | Universal Data Graph / GraphQL federation; Tyk OAS (OpenAPI-native) introduced (4.1). |
| 5.0 | 2023-05 | Major release; `ee/` commercial directory and dual-licensing in-tree[^2]. |
| 5.x | 2024–2026 | Ongoing OAS feature parity, streaming (async APIs), continued releases. |

## References

[^1]: Tyk — company and project background, open-sourced 2014. https://tyk.io/about/
[^2]: Tyk licensing — root code under MPL-2.0, `ee/` directory under a commercial license. https://github.com/TykTechnologies/tyk/blob/master/LICENSE

## Tags

go, api-gateway, api-management, reverse-proxy, rate-limiting, graphql, grpc, kubernetes, microservices, redis, cloud-native, oss-with-commercial
