# traefik/traefik

> A reverse proxy and load balancer that discovers its own routes from your orchestrator instead of being configured file-by-file.

[GitHub repo](https://github.com/traefik/traefik) ·
[Official website](https://traefik.io) ·
[License: MIT](https://github.com/traefik/traefik/blob/master/LICENSE.md)

## Overview

Traefik is an HTTP/TCP/UDP reverse proxy and load balancer written in Go, first released by Emile Vauge in 2016 and now maintained by Traefik Labs (formerly Containous)[^1]. Its defining idea is *auto-discovery*: instead of hand-writing a route block per service, Traefik watches a "provider" — the Docker socket, the Kubernetes API, Consul, etcd, a file — and generates its routing table live as services appear and disappear. Pointing it at an orchestrator is meant to be the only configuration step for the common case.

With ~64k stars and active daily commits[^2], Traefik is one of the two or three default ingress choices in the cloud-native world alongside NGINX and Envoy. It occupies a specific niche: dynamic, label/annotation-driven configuration with built-in Let's Encrypt, a web dashboard, and a single-binary deployment. The tradeoff is that this convenience is coupled to Traefik's own configuration model, which is unusual, has changed shape twice (v1→v2→v3), and pushes real complexity into how you split *static* from *dynamic* config.

The project is open-source MIT, but sits under a commercial company whose paid products — Traefik Enterprise and the Traefik Hub API gateway — provide the high-availability and API-management features that the OSS proxy deliberately leaves thin. Understanding which capabilities live on which side of that line is the main thing operators get wrong.

## Getting Started

```bash
# Run the official image against the Docker provider
docker run -d -p 80:80 -p 8080:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  traefik:v3.3 --api.insecure=true --providers.docker=true
```

Traefik reads routing intent from labels on your other containers:

```yaml
# docker-compose.yml — Traefik configures itself from these labels
services:
  traefik:
    image: traefik:v3.3
    command:
      - --providers.docker=true
      - --entrypoints.web.address=:80
    ports: ["80:80"]
    volumes: ["/var/run/docker.sock:/var/run/docker.sock:ro"]

  whoami:
    image: traefik/whoami
    labels:
      - traefik.http.routers.whoami.rule=Host(`whoami.localhost`)
      - traefik.http.routers.whoami.entrypoints=web
```

No proxy restart is needed when `whoami` starts, stops, or scales — Traefik reconciles the change from the Docker event stream.

## Architecture / How It Works

The request pipeline is four staged concepts, and getting them straight is most of the learning curve:

1. **EntryPoints** — the TCP/UDP ports Traefik listens on (`:80`, `:443`). Defined in *static* config only.
2. **Routers** — match incoming requests by rule (`Host()`, `PathPrefix()`, `Headers()`, and boolean combinations) and hand them to a service.
3. **Middlewares** — an ordered chain between router and service: `stripPrefix`, `headers`, `rateLimit`, `basicAuth`, `redirectScheme`, `ipAllowList`, etc. Middlewares are composable and reusable across routers.
4. **Services** — the load-balancer definition: the actual backend servers, health checks, sticky sessions, and balancing.

The critical split is **static vs dynamic configuration**. Static config (entrypoints, which providers are enabled, ACME resolvers, the API/dashboard) is read once at startup from CLI flags, environment variables, or a `traefik.yml`/`.toml` file — changing it requires a restart. Dynamic config (routers, middlewares, services, TLS) is hot-reloaded from providers with no restart. These two layers use different syntax and different sources, and conflating them is the single most common source of confusion.

**Providers** are the discovery backends. Each translates its native model into Traefik's dynamic config: Docker/Swarm labels, the File provider (a watched YAML/TOML file for manual routes), the KV stores (Consul, etcd), and Kubernetes — where Traefik supports the standard Ingress resource, its own `IngressRoute` CRD (the fullest-featured path), and the Gateway API. Multiple providers can run at once and their outputs merge.

**TLS / ACME** is built in: Traefik can obtain and renew Let's Encrypt certificates via HTTP-01, TLS-ALPN-01, or DNS-01 challenges, storing them in an `acme.json` file. **Plugins** (added in v2, made first-class in v3) run as WebAssembly modules via the Yaegi interpreter or the Wazero WASM runtime, letting you add middleware without recompiling the binary.

## Production Notes

**The ACME/HA gap is the biggest footgun.** OSS Traefik stores certificates in a local `acme.json` per instance. Running multiple replicas behind the same Let's Encrypt account will cause the instances to race on issuance and hit rate limits, because there is no built-in certificate store synchronization — that is a Traefik Enterprise feature. The standard OSS workarounds are: run a single Traefik instance for ACME, use the DNS-01 challenge with a shared/external store, or delegate certificate management entirely to `cert-manager` and let Traefik consume the resulting secrets. `acme.json` must also be `chmod 600` or Traefik refuses to start.

**Static vs dynamic mistakes.** Trying to define entrypoints in dynamic config, or routers in the static file, silently does nothing. A large fraction of "my route isn't working" issues trace back to config placed in the wrong layer.

**Migrations are real work.** v1→v2 (2019) was a near-total rewrite of the configuration model — frontends/backends became routers/services/middlewares, and there was no automatic translation. v2→v3 (2024) was smaller but still breaking: rule syntax changed (`PathPrefix` matcher semantics, backtick-quoted values), several legacy providers were removed (Marathon, Mesos, Rancher v1, the old Kubernetes Ingress annotations were tightened), and some middleware options were renamed[^3]. Always read the migration guide before a major bump; pin the minor version in image tags (`traefik:v3.3`, not `traefik:latest`).

**Dashboard exposure.** `--api.insecure=true` publishes the dashboard on `:8080` with no auth — fine for a laptop, a live security hole in production. Secure it behind a router with `basicAuth`/`forwardAuth` and TLS, or disable it.

**Performance.** Traefik is fast enough for the overwhelming majority of workloads, but in head-to-head synthetic proxy benchmarks it generally trails tuned NGINX and HAProxy on raw requests-per-second and tail latency; the Go GC and its richer per-request middleware pipeline cost some throughput. If you are proxying at the very top of the scale envelope, benchmark against your own traffic rather than trusting either side's marketing.

**Observability.** Prometheus, Datadog, StatsD, and InfluxDB metrics are built in, and v3 moved tracing to native OpenTelemetry. Access logs support JSON and CLF. These are solid, but per-service metric cardinality can explode in large clusters — scope your labels.

## When to Use / When Not

**Use when:**
- You run Docker or Kubernetes and want routes to appear automatically from labels/CRDs with no proxy restarts.
- You want built-in, automatic Let's Encrypt certificates for a small-to-medium setup.
- You value a single Go binary, a dashboard, and a gentle happy path over maximum tunability.
- You want middleware composition (auth, rate limiting, header rewriting) declared next to your services.

**Avoid when:**
- You need highly-available automatic TLS across many replicas without buying Traefik Enterprise or bolting on cert-manager.
- You are chasing absolute peak throughput / lowest tail latency — tuned NGINX, HAProxy, or Envoy will edge it out.
- You want a mature service-mesh data plane — that is Envoy's domain, not Traefik's.
- Your team already has deep NGINX operational expertise and no need for dynamic discovery.

## Alternatives

- envoyproxy/envoy — use when you need a programmable L7 proxy or a service-mesh data plane (Istio, Consul) rather than a self-configuring edge router.
- kubernetes/ingress-nginx — use when you want the battle-tested default Kubernetes ingress and NGINX operational familiarity.
- caddyserver/caddy — use when you want automatic HTTPS on a simpler single-server or small-fleet setup with a Caddyfile.
- haproxy/haproxy — use when raw TCP/HTTP performance, load-balancing precision, and fine tuning control matter most.
- nginx/nginx — use when you want a static, hand-configured, extensively documented reverse proxy without dynamic discovery.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-05 | First stable release. Frontend/backend model, initial provider auto-discovery[^1]. |
| 1.7 | 2018-09 | Final v1 line; long-lived, widely deployed. |
| 2.0 | 2019-09 | Rewritten config model: routers / middlewares / services, TCP support, IngressRoute CRD[^4]. |
| 2.5 | 2021-08 | HTTP/3 (experimental), Kubernetes Gateway API provider, plugin ecosystem growth. |
| 3.0 | 2024-04 | WASM plugins via Wazero, native OpenTelemetry, rule-syntax changes, legacy providers removed[^3]. |
| 3.3 | 2025 | Current v3 line; iterative provider and middleware improvements. |

## References

[^1]: Traefik Labs, project history and about page. https://traefik.io/traefik/
[^2]: GitHub repository (stars, forks, activity), fetched 2026-07. https://github.com/traefik/traefik
[^3]: Traefik documentation, "Migration Guide: v2 to v3". https://doc.traefik.io/traefik/migrate/v2-to-v3/
[^4]: Traefik Labs blog, "Traefik 2.0" announcement (2019). https://traefik.io/blog/

## Tags

go, reverse-proxy, load-balancer, kubernetes, docker, ingress, cloud-native, lets-encrypt, api-gateway, microservices
