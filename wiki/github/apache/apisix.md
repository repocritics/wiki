# apache/apisix

> A dynamic, hot-reloadable API gateway built on OpenResty (NGINX + LuaJIT) with etcd as its configuration store.

[GitHub repo](https://github.com/apache/apisix) ·
[Official website](https://apisix.apache.org) ·
[License: Apache-2.0](https://github.com/apache/apisix/blob/master/LICENSE)

## Overview

APISIX is an API gateway and reverse proxy that sits at the edge of a service
mesh or microservice deployment, handling routing, load balancing,
authentication, rate limiting, observability, and traffic shaping through a
plugin system. It was open-sourced by the Chinese company Zhiliu Technology
(now API7.ai) in 2019, entered the Apache incubator in October 2019, and
graduated to a top-level Apache project in July 2020[^1]. As of 2026 it is one
of the most-starred open-source API gateways (~16.9k stars) and the primary
open-source competitor to Kong.

The defining architectural choice is that APISIX stores **all** runtime
configuration — routes, upstreams, plugins, certificates, consumers — in
**etcd**, and every gateway worker watches etcd for changes and applies them in
memory without a reload or restart. This is what "dynamic" and "hot-reload"
mean in APISIX's marketing: adding a route or flipping a plugin takes effect in
milliseconds across the cluster. The tradeoff is that etcd becomes a hard,
first-class operational dependency: your gateway's control plane is only as
healthy as your etcd cluster, and running etcd well is a non-trivial skill in
its own right.

Since roughly 2024 the project has repositioned around an **"AI Gateway"**
story — proxying and load-balancing across LLM providers, token-based rate
limiting, and an `mcp-bridge` plugin that fronts MCP servers[^2]. This is a
plugin layer on the same core, not a separate product.

## Getting Started

Quickstart via the vendor's install script (requires Docker):

```shell
curl -sL https://run.api7.ai/apisix/quickstart | sh
```

This boots APISIX on port `9080` (data plane) plus an etcd instance. The Admin
API listens on `9180`. Create a route that proxies to an upstream:

```shell
curl -i "http://127.0.0.1:9180/apisix/admin/routes/1" -X PUT \
  -H "X-API-KEY: <admin-key>" -d '
{
  "uri": "/get",
  "upstream": {
    "type": "roundrobin",
    "nodes": { "httpbin.org:80": 1 }
  }
}'

curl "http://127.0.0.1:9080/get"    # flows through the gateway
```

All configuration is data pushed to the Admin API (which writes to etcd), not
files you edit and reload — with the exception of standalone mode (below).

## Architecture / How It Works

APISIX is an OpenResty application: NGINX with LuaJIT embedded. Each NGINX
worker process runs the same Lua code and holds an in-memory copy of the
configuration it has read from etcd.

- **Config store (etcd).** On startup and continuously thereafter, workers
  `watch` etcd v3 (over its gRPC-gateway HTTP interface) for changes to
  `/routes`, `/upstreams`, `/services`, `/plugins`, `/ssl`, `/consumers`, etc.
  Changes propagate to every worker without a reload.
- **Router (radixtree).** Route matching uses `lua-resty-radixtree`, a radix
  tree over URI/host/method plus arbitrary NGINX variables and operator
  expressions, with a priority field to disambiguate overlaps[^3].
- **Plugin phases.** A plugin hooks one or more request phases — `rewrite`,
  `access`, `header_filter`, `body_filter`, `log`, and the `balancer` stage.
  Plugins have numeric priorities that determine execution order within a
  phase. ~100 plugins ship in-tree (auth, rate limiting, observability,
  transformation, serverless, AI proxying).
- **Admin API vs data plane.** Port `9180` mutates config; port `9080`/`9443`
  serves traffic. In open-source APISIX these run in the same process — there is
  no built-in control-plane/data-plane separation (that is an API7 commercial
  feature).
- **External plugin runners.** Plugins can be written in Java, Go, Python, or
  Node.js and run as a **sidecar process** that APISIX talks to over local RPC
  per request. This avoids writing Lua but adds per-request IPC latency. There
  is also an experimental Wasm path via the proxy-wasm SDK.
- **Standalone mode.** APISIX can load routes from a local YAML file instead of
  etcd, which suits GitOps and Kubernetes workflows where you don't want a
  stateful config store. In this mode the Admin API is read-only.

The companion `apisix-ingress-controller` project lets APISIX act as a
Kubernetes ingress, translating CRDs into APISIX config.

## Production Notes

**etcd is the operational center of gravity.** This is the single most common
source of production pain. You must run and maintain an etcd cluster (odd
number of nodes, TLS, backups). Watch for the default 2 GB etcd storage quota,
which triggers an alarm that silently makes the keyspace read-only until you
compact and defragment — at which point config changes stop applying. Network
partitions between APISIX and etcd mean workers keep serving the last-known
config but reject updates.

**Change the default admin key.** Older releases shipped a well-known default
Admin API key (`edd1c9f034335f136f87ad84b625c8f1`) and bound the Admin API
loosely. Leaving the default key or exposing port `9180` beyond `127.0.0.1` is
a real-world compromise vector; lock down `allow_admin` and rotate the key
before anything faces a network.

**You are writing Lua.** Custom in-tree plugins and any deep debugging require
LuaJIT familiarity — a smaller talent pool than Go/Java gateways. The external
plugin runners mitigate this but at a latency cost, so hot-path logic still
tends to end up in Lua.

**2.x → 3.x was a breaking migration.** APISIX 3.0 (late 2022) changed
configuration structure and defaults (deployment roles, the standalone/config
format, etcd interaction) enough that upgrades were not drop-in; read the
migration notes and test config compatibility rather than assuming a rolling
upgrade. Minor releases are generally smoother.

**Sizing shared memory.** Rate-limiting counters, plugin state, and caches live
in NGINX shared dictionaries (`lua_shared_dict`). Under-sizing these surfaces as
eviction and inaccurate limit counts under load rather than a clean error.

**Performance is genuinely high** — single-core throughput in the tens of
thousands of QPS with sub-millisecond added latency is achievable for simple
routes[^4] — but real numbers collapse as you stack plugins (especially
body-buffering transforms, external auth, and logging plugins that make network
calls in the request path). Benchmark with your actual plugin chain, not a bare
route.

## When to Use / When Not

**Use when:**
- You want dynamic, restart-free config changes at scale and can operate etcd.
- You need a broad in-tree plugin catalog (auth, rate limiting, tracing,
  transforms, LLM proxying) without buying an enterprise tier.
- You're already an OpenResty/NGINX shop and value that performance profile.
- You want an Apache-governed project with no single-vendor license risk.

**Avoid when:**
- You don't want to run etcd and standalone/YAML mode doesn't fit your workflow.
- Your team has no Lua/OpenResty capacity and you need heavy custom logic on the
  hot path.
- You want built-in control-plane/data-plane separation, multi-tenant RBAC, or a
  polished management UI out of the box (these lean commercial or require
  assembling `apisix-dashboard` and other pieces).
- A managed cloud gateway (AWS API Gateway, GCP API Gateway) already covers your
  needs and you'd rather not operate a gateway at all.

## Alternatives

- Kong/kong — the incumbent OpenResty-based gateway; APISIX's most direct
  comparison. Use Kong when you want a larger commercial ecosystem and are fine
  with its Postgres/DB-less config model instead of etcd.
- envoyproxy/envoy — lower-level L4/L7 proxy and the data plane behind most
  service meshes; use Envoy (often via a control plane) when you need mesh-grade
  traffic control rather than a batteries-included API gateway.
- traefik/traefik — Go gateway with first-class dynamic service discovery; use
  Traefik when you want tight Docker/Kubernetes label-driven config and Go
  extensibility.
- emissary-ingress/emissary — Envoy-based Kubernetes-native API gateway; use it
  when your world is entirely Kubernetes CRDs.
- Tyk (TykTechnologies/tyk) — Go gateway with a strong built-in dashboard and
  API management layer; use Tyk when integrated management UI matters more than
  raw OpenResty performance.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2019-06 | Initial open-source release by Zhiliu/iresty[^1]. |
| — | 2019-10 | Entered the Apache Incubator[^1]. |
| — | 2020-07 | Graduated to Apache top-level project[^1]. |
| 2.0 | 2020-11 | etcd v3 config store, plugin and schema maturation. |
| 3.0 | 2022-12 | Major release; config/deployment restructure, breaking upgrade[^5]. |
| AI Gateway | 2024–2025 | LLM proxy, token rate limiting, `mcp-bridge` plugins[^2]. |

## References

[^1]: Apache Software Foundation, "Apache APISIX" project page and podling
history. https://apisix.apache.org/ and
https://incubator.apache.org/projects/apisix.html
[^2]: Apache APISIX, "AI Gateway" and "Host MCP Server with API Gateway"
(2025-04-21). https://apisix.apache.org/ai-gateway/ and
https://apisix.apache.org/blog/2025/04/21/host-mcp-server-with-api-gateway/
[^3]: `lua-resty-radixtree` — the radix-tree router used by APISIX.
https://github.com/api7/lua-resty-radixtree
[^4]: Apache APISIX benchmark documentation and script.
https://apisix.apache.org/docs/apisix/benchmark/
[^5]: Apache APISIX 3.0 release notes / upgrade guide.
https://apisix.apache.org/docs/apisix/CHANGELOG/

## Tags

lua, luajit, openresty, nginx, api-gateway, reverse-proxy, ai-gateway, etcd, cloud-native, kubernetes-ingress, load-balancing, apache
