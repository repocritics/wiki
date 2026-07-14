# caddyserver/caddy

> An extensible Go web server that obtains and renews TLS certificates automatically, configured by a single JSON document.

[GitHub repo](https://github.com/caddyserver/caddy) ·
[Official website](https://caddyserver.com) ·
[License: Apache-2.0](https://github.com/caddyserver/caddy/blob/master/LICENSE)

## Overview

Caddy is an HTTP/1-2-3 web server and reverse proxy written in Go, first released by Matthew Holt in 2015 while he was a student at Brigham Young University[^1]. Its defining feature is **automatic HTTPS**: on startup Caddy inspects its configured site addresses, provisions certificates from an ACME CA (Let's Encrypt or ZeroSSL for public names, a built-in local CA for internal names), and renews them in the background — no cron jobs, no certbot, no manual `ssl_certificate` directives. It was the first server to do this by default.

The project underwent a hard break at **version 2.0 (2020)**, which was a ground-up rewrite[^2]. Caddy 1.x was a monolithic server driven by the Caddyfile; Caddy 2.x is a modular platform whose canonical configuration is a single JSON document, with the Caddyfile demoted to one of several "config adapters" that compile down to that JSON. This rewrite broke essentially all v1 configs and plugins, and the two versions share little beyond the name. Most material written before mid-2020 refers to an architecture that no longer exists — a persistent source of confusion for newcomers.

The central tension in Caddy is convenience versus control. The Caddyfile makes a working HTTPS reverse proxy a two-line file, but the real configuration surface is the JSON API, and non-trivial setups eventually require understanding both layers. Plugins are compiled in, not loaded dynamically, so extending Caddy means rebuilding the binary.

## Getting Started

```bash
# Debian/Ubuntu (official apt repo), or download a static binary from Releases
sudo apt install caddy
# ad-hoc reverse proxy with automatic HTTPS, no config file:
caddy reverse-proxy --from example.com --to localhost:8080
```

A minimal `Caddyfile` — this alone provisions and renews a real certificate:

```caddyfile
example.com {
    encode gzip
    reverse_proxy localhost:8080
}

files.example.com {
    root * /var/www/files
    file_server browse
}
```

```bash
caddy run --config ./Caddyfile        # foreground
caddy reload --config ./Caddyfile     # graceful zero-downtime reload
```

## Architecture / How It Works

Caddy is best understood as a **module host**, not a web server that happens to have plugins. Everything — HTTP handlers, TLS issuers, storage backends, loggers, config adapters — is a module registered against a namespaced ID (`http.handlers.reverse_proxy`, `tls.issuance.acme`, `caddy.storage.file_system`). The JSON config is a tree that names modules and supplies their raw settings; at load time Caddy unmarshals each subtree directly into the corresponding Go struct. You are, quite literally, setting in-memory field values of the running program[^3].

Key pieces:

- **Config layer.** JSON is native and authoritative. A **config adapter** turns another format (Caddyfile, YAML, TOML, NGINX config) into that JSON before load. The Caddyfile is convenient but strictly less expressive than the JSON it produces; `caddy adapt` shows the compiled output.
- **Admin API.** A REST API (default `localhost:2019`) is the real control plane. `caddy reload` is a client of it. Config changes are applied atomically with graceful connection draining, so reloads drop no requests.
- **CertMagic.** Certificate automation lives in a separate library, `caddyserver/certmagic`, which Caddy embeds[^4]. It handles ACME order/challenge/renewal, OCSP stapling, and on-disk (or pluggable) storage of keys and certs.
- **Transport.** HTTP/1.1, HTTP/2, and HTTP/3 (QUIC over UDP) are all served by default on the same listener.

Because modules are wired at compile time, adding a plugin means producing a new binary. The supported path is **`xcaddy`**, a build tool that generates a `main.go` importing your chosen plugins and runs `go build`. There is no runtime `LoadModule` / `.so` mechanism.

## Production Notes

**Plugins require rebuilds.** The stock binary ships a limited module set. Adding `caddy-dns/cloudflare` (needed for wildcard/DNS-01 certs behind a firewall) or any third-party handler means `xcaddy build --with ...` and redeploying. Teams often forget this and pin a hand-built binary that then drifts from upstream releases. `caddy list-modules` shows what a given binary actually contains.

**HTTP/3 and UDP buffers.** With HTTP/3 on by default, Caddy commonly logs `failed to sufficiently increase receive buffer size` on Linux. It is a real throughput warning, fixed by raising `net.core.rmem_max` / `wmem_max` via sysctl[^5]. It is not fatal but is one of the most-asked-about log lines.

**Clustering needs shared storage.** Automatic HTTPS state (account keys, certificates, locks) is written to a storage backend — by default the local filesystem under the data directory. Running multiple Caddy instances for the same domains **without** a shared storage module (e.g. Redis, Consul, Postgres, or a shared volume) causes duplicate ACME orders and can trip Let's Encrypt rate limits. Point all instances at one storage backend.

**The admin API is a footgun if exposed.** `localhost:2019` is unauthenticated by design and grants full config control. It must never be bound to a public interface without additional protection; leaving it open is a full compromise of the server's routing.

**ACME rate limits.** During iteration, hitting the production Let's Encrypt CA repeatedly can exhaust the per-domain limit. Use the staging CA (`acme_ca` pointing at the staging endpoint) while testing, then switch back.

**Config-format habits.** Because the Caddyfile hides the JSON, some behaviors (handler ordering, matcher precedence) are surprising until you run `caddy adapt` and read the generated tree. For anything beyond a straight reverse proxy, treat the JSON as the source of truth and the Caddyfile as sugar.

**Upgrade story.** Within v2, upgrades are generally smooth — a single static binary swap with no libc dependency. The one-time cost was v1 → v2, which was a full config and plugin rewrite with no automated migration path[^2].

## When to Use / When Not

**Use when:**
- You want TLS to be automatic and correct without managing certbot or renewal cron.
- You're fronting a handful to thousands of sites and value a single small static binary with no runtime dependencies.
- You want on-the-fly, zero-downtime config changes via an API.
- You're serving internal services and want a local CA for `*.internal` names.

**Avoid when:**
- You need dynamic, label-driven service discovery as the primary model (Kubernetes ingress, Docker autodiscovery) — a proxy built around that fits better out of the box.
- You have deep existing investment in nginx/Apache config and modules and no automatic-HTTPS pain to solve.
- You need to add many third-party modules but cannot run a custom build pipeline — the compile-in model becomes operational friction.
- You need raw L4/TCP load balancing at extreme connection counts as the core workload.

## Alternatives

- traefik/traefik — cloud-native reverse proxy with dynamic Docker/Kubernetes service discovery; use it when routing is driven by container labels rather than a static config.
- nginx/nginx — battle-tested, maximal raw throughput, manual TLS; use it when you already have nginx config and don't need automatic HTTPS.
- envoyproxy/envoy — L7 proxy for service meshes with advanced traffic management and observability; use it at large scale where Caddy's simplicity is not the priority.
- haproxy/haproxy — high-performance load balancer strong at L4/TCP; use it for connection-heavy load balancing rather than site serving.
- apache/httpd — mature module ecosystem and `.htaccess`; use it for legacy or shared-hosting environments built around those conventions.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.8 | 2016-01 | First release to enable automatic HTTPS by default[^1]. |
| 1.0 | 2019-05 | First stable release; end of the monolithic Caddyfile-driven era. |
| 2.0 | 2020-05 | Ground-up rewrite: modular platform, native JSON config, admin API[^2]. |
| 2.1 | 2020-06 | HTTP/3 (experimental), ZeroSSL added as a default issuer. |
| 2.6 | 2022-09 | HTTP/3 enabled by default; on-demand TLS improvements. |
| 2.7 | 2023-06 | Config and FastCGI/named-matcher enhancements. |
| 2.8 | 2024-05 | Encrypted ClientHello (ECH) groundwork, logging refinements. |
| 2.9 | 2025-01 | Continued module and TLS updates on the v2 line. |

## References

[^1]: Caddy "About" — project origin (Matthew Holt) and first automatic-HTTPS-by-default server. https://github.com/caddyserver/caddy#about
[^2]: Matthew Holt, "Announcing Caddy 2" — the v2 rewrite and JSON/modular architecture. https://caddyserver.com/v2
[^3]: Caddy docs, "Architecture" — modules, config structure, and how JSON maps to in-memory types. https://caddyserver.com/docs/architecture
[^4]: CertMagic — the automatic-TLS library Caddy embeds. https://github.com/caddyserver/certmagic
[^5]: quic-go, "UDP Buffer Sizes" — cause and sysctl fix for the buffer-size warning Caddy surfaces. https://github.com/quic-go/quic-go/wiki/UDP-Buffer-Sizes

## Tags

go, web-server, reverse-proxy, automatic-https, tls, acme, http3, self-hosting, devops, static-binary
