# Kong/kong

> A Lua/OpenResty-based API gateway that proxies, authenticates, and rate-limits service traffic through a plugin pipeline — now repositioned around LLM and MCP traffic.

[GitHub repo](https://github.com/Kong/kong) ·
[Official website](https://konghq.com/install/) ·
[License: Apache-2.0](https://github.com/Kong/kong/blob/master/LICENSE)

## Overview

Kong is an API gateway that sits in front of your services and applies cross-cutting concerns — routing, load balancing, authentication, rate limiting, request/response transformation, logging — as configurable plugins rather than application code. It was open-sourced by Mashape in 2015 and reached a 1.0 release in 2018[^1]. As of 2026 it is one of the most widely deployed open-source gateways, with a large commercial business (Kong Inc., Konnect SaaS) built around the same core.

The defining architectural fact is that Kong is built on **OpenResty** — nginx compiled with an embedded LuaJIT interpreter[^2]. Kong is, in effect, a large body of Lua that runs inside nginx worker processes. This is the source of both its performance (nginx's event loop does the heavy proxying) and its extensibility (plugins are Lua modules that hook into request/response phases). It is also the source of its constraints: the plugin model, the concurrency model, and the performance envelope are all nginx's.

Since roughly 2024 Kong has repositioned itself as an "API 𖧹 LLM 𖧹 MCP" gateway. The GitHub description is now "The API and AI Gateway," and a family of `ai-*` plugins route and govern traffic to OpenAI, Anthropic, Bedrock, Gemini, and others through a common interface[^3]. This is a genuine feature line, not just marketing, but the underlying gateway is the same OpenResty core it has always been — the AI features are plugins.

## Getting Started

The quickest local setup is the docker-compose distribution (Postgres-backed):

```bash
git clone https://github.com/Kong/docker-kong
cd docker-kong/compose/
KONG_DATABASE=postgres docker-compose --profile database up
```

Kong then exposes the proxy on `:8000` and the Admin API on `:8001`. Configure a service and route, then send traffic through the gateway:

```bash
# Register an upstream service
curl -s -X POST http://localhost:8001/services \
  --data name=example \
  --data url=http://httpbin.org

# Attach a route to it
curl -s -X POST http://localhost:8001/services/example/routes \
  --data 'paths[]=/example'

# Proxy a request through Kong
curl -i http://localhost:8000/example/get

# Enable a plugin on the route (e.g. rate limiting)
curl -s -X POST http://localhost:8001/services/example/plugins \
  --data name=rate-limiting \
  --data config.minute=5
```

For production, the declarative (DB-less) path uses a single YAML file managed with [decK](https://github.com/kong/deck) instead of imperative Admin API calls.

## Architecture / How It Works

Kong runs as an nginx master plus N worker processes. Each incoming request passes through nginx phases (`access`, `header_filter`, `body_filter`, `log`), and Kong invokes enabled plugins at each phase in priority order. A plugin is a Lua table implementing phase handler functions plus a schema; the **Plugin Development Kit (PDK)** gives it a stable API for reading/mutating the request, calling upstream, and emitting logs[^4].

Configuration entities — services, routes, consumers, plugins, upstreams — can be stored three ways, and the choice dictates the whole operational model:

1. **Traditional (DB-backed)** — entities live in PostgreSQL. The Admin API mutates the database; every node polls it. Historically Cassandra was also supported, but it was deprecated and removed during the 3.x series; Postgres is the only supported datastore now.
2. **DB-less / declarative** — no database. The entire config is loaded from a YAML/JSON file into each node's memory at boot. Immutable at runtime except by reloading the file. This is the common Kubernetes and GitOps pattern.
3. **Hybrid (control plane / data plane)** — a control-plane node owns the database and Admin API; stateless data-plane nodes receive config over a TLS channel and proxy traffic. This decouples config management from the request path and is the recommended model for scale.

Routing was rewritten in Kong 3.0 around an expressions-based router (the "ATC" router) that compiles match rules; the older regex/priority router remains available via a compatibility flag[^5]. On Kubernetes, Kong is driven by the separate **Kubernetes Ingress Controller** repo, which translates Ingress/Gateway API resources into Kong config.

Plugins can also be written in **Go** or **JavaScript** via an external plugin server (a sidecar process Kong talks to over a socket), trading in-process Lua performance for language choice.

## Production Notes

**The database is the historical footgun.** In traditional mode every node polls Postgres, and the DB becomes a shared point of failure and a scaling ceiling. Large deployments are strongly steered toward hybrid or DB-less mode precisely to take the database off the request path. If you run traditional mode, the Postgres instance needs the same HA rigor as any critical datastore.

**Plugin ordering and cost are not obvious.** Plugins run in priority order, and an expensive plugin (external auth, heavy transformation, rate-limiting backed by Redis) on a hot route is charged per request inside the nginx worker. A slow plugin blocks the worker's cooperative scheduling. Profile the plugin chain on hot paths; the gateway's own overhead is usually small compared to what you bolt onto it.

**Rate limiting has a policy tradeoff.** The `rate-limiting` plugin's `local` policy is fast but per-node (so N nodes allow N× the limit); `cluster`/`redis` policies are globally accurate but add a network hop and a shared dependency. Picking the wrong policy silently either over-permits or adds latency.

**Upgrades across major versions require care.** The 2.x → 3.x jump changed the router, config format expectations, and deprecated datastores; declarative configs and custom plugins frequently need adjustment. Read the per-version upgrade and breaking-changes notes rather than bumping the image tag blindly[^6].

**Custom Lua plugins couple you to internals.** The PDK is stable, but plugins that reach past it into Kong or ngx internals break across versions. Keep custom plugins on the PDK surface.

**Open source vs. Konnect.** Many operational conveniences (managed control plane, analytics, dev portal, some plugins) are part of the commercial Konnect/Enterprise offering. The OSS gateway is fully functional, but evaluate which features you assume are free actually are before committing.

## When to Use / When Not

**Use when:**
- You need a plugin-driven gateway in front of many services and want auth, rate limiting, and transforms as config, not code.
- You want DB-less/declarative config that fits GitOps, or hybrid mode for control-plane/data-plane separation.
- You're on Kubernetes and want Ingress/Gateway API support with a mature plugin ecosystem.
- You want a single ingress point that can also govern LLM/MCP traffic across providers.

**Avoid when:**
- You want a service-mesh data plane with xDS-driven config — Envoy is the better fit.
- You want the lightest possible Kubernetes ingress with zero external moving parts — Traefik or plain nginx ingress is simpler.
- Your team has no appetite for Lua and you expect to write substantial custom gateway logic.
- You need only a plain reverse proxy; Kong's entity/plugin model is overhead you won't use.

## Alternatives

- envoyproxy/envoy — use when you want a programmable L7 proxy / service-mesh data plane configured via xDS rather than a plugin-and-entity API gateway.
- apache/apisix — use when you want a very similar OpenResty/Lua architecture but with etcd-based config and Apache Foundation governance.
- traefik/traefik — use when you want Go, automatic service discovery, and a lighter Kubernetes-native ingress with fewer moving parts.
- TykTechnologies/tyk — use when you want a Go-extensible API gateway instead of writing plugins in Lua.
- nginx / nginxinc/kubernetes-ingress — use when you need a plain reverse proxy or basic ingress without a gateway plugin ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015 | Open-sourced by Mashape; OpenResty-based gateway[^1]. |
| 1.0 | 2018-09 | First stable major; plugin API and Admin API stabilized[^1]. |
| 2.0 | 2020-01 | Go plugin support, hybrid mode groundwork, deprecations cleaned up. |
| 3.0 | 2022-09 | New expressions/ATC router, config and schema changes, Cassandra deprecation path[^5]. |
| 3.4 | 2023 | Cassandra support removed; Postgres-only datastore. |
| 3.6+ | 2024 | AI Gateway plugin family (`ai-proxy` and related) for multi-LLM routing[^3]. |
| 3.x | 2026-07 | Ongoing 3.x line; MCP gateway and AI traffic features expanded[^3]. |

## References

[^1]: Kong Gateway (OSS) repository and release history. https://github.com/Kong/kong/releases
[^2]: OpenResty — nginx + LuaJIT, the runtime Kong is built on. https://openresty.org/
[^3]: Kong AI Gateway documentation. https://developer.konghq.com/ai-gateway/
[^4]: Kong Plugin Development Kit (PDK) reference. https://docs.konghq.com/gateway/latest/plugin-development/pdk/
[^5]: Kong Gateway 3.0 router and breaking changes. https://docs.konghq.com/gateway/changelog/
[^6]: Kong Gateway upgrade guide. https://docs.konghq.com/gateway/latest/upgrade/

## Tags

lua, openresty, api-gateway, reverse-proxy, kubernetes-ingress, rate-limiting, ai-gateway, llm-gateway, microservices, cloud-native, plugins
