# apache/httpd

> The Apache HTTP Server — the modular, config-file-driven web server that ran most of the web for two decades and still anchors a large share of it.

[GitHub repo](https://github.com/apache/httpd) ·
[Official website](https://httpd.apache.org) ·
[License: Apache-2.0](https://github.com/apache/httpd/blob/trunk/LICENSE)

## Overview

Apache httpd is an HTTP/1.1 and HTTP/2 web server written in C, in continuous development since 1995[^1]. It was the founding project of the Apache Software Foundation and, for most of the 2000s and early 2010s, the single most-deployed web server on the internet. Its defining trait is a module architecture: a small core plus a large set of loadable modules (`mod_ssl`, `mod_rewrite`, `mod_proxy`, `mod_headers`, and dozens more) that add nearly all functionality. Configuration is file-driven — `httpd.conf` plus per-directory `.htaccess` overrides — rather than programmatic.

The GitHub repository is a read-only mirror; development actually happens in Apache's own Subversion/git infrastructure, and issues are tracked in Bugzilla, not GitHub Issues[^1]. Stars (~4.1k) and forks therefore badly understate the project's real footprint — this is infrastructure that predates GitHub and never migrated its workflow there.

The central tension in 2026 is age versus incumbency. Apache lost its performance and reverse-proxy lead to nginx over the late 2010s[^2], and greenfield deployments increasingly start elsewhere. But it remains deeply entrenched: the default in cPanel/WHM hosting, the reference server for `.htaccess`-based shared hosting, and the path of least resistance for classic PHP applications via `mod_php`. It is rarely the fastest choice and frequently the most compatible one.

## Getting Started

```bash
# Debian/Ubuntu
sudo apt install apache2
# RHEL/Fedora
sudo dnf install httpd
# macOS (Homebrew)
brew install httpd
```

A minimal virtual host serving static files with a reverse proxy to a backend app:

```apache
# /etc/apache2/sites-available/example.conf  (or conf.d/*.conf)
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/example

    <Directory /var/www/example>
        Require all granted
        Options -Indexes
    </Directory>

    # Proxy /api to a local app server
    ProxyPass        /api  http://127.0.0.1:3000/
    ProxyPassReverse /api  http://127.0.0.1:3000/
</VirtualHost>
```

```bash
sudo a2enmod proxy proxy_http    # load modules (Debian helper)
sudo apachectl configtest        # validate config before reload
sudo systemctl reload apache2
```

## Architecture / How It Works

The core is deliberately small; behavior comes from modules and from the **request processing phases**. Each incoming request passes through an ordered pipeline — URI translation, access control, authentication, authorization, MIME-type checks, the content handler, and logging — and modules register hooks into whichever phases they care about. `mod_rewrite`, for example, hooks the translation phase; `mod_ssl` wraps the connection layer.

**Multi-Processing Modules (MPMs)** decide the concurrency model, and only one is active per build:

- **prefork** — one request per process, no threads. Maximally compatible with non-thread-safe libraries (historically the safe choice for `mod_php`), but memory-heavy under load.
- **worker** — hybrid: multiple processes, each with multiple threads.
- **event** — worker plus a dedicated listener thread that keeps idle keep-alive connections off worker threads. The default MPM since 2.4[^3] and the right choice for most modern deployments.

Underneath sits the **Apache Portable Runtime (APR)**, a separate library that abstracts OS primitives (sockets, memory pools, file I/O, threads) so httpd builds across Unix and Windows. Memory is managed through APR **pools** — arena allocators tied to a request or connection lifetime, freed all at once rather than object by object.

The architectural consequence worth internalizing: Apache's model is one worker occupied per in-flight request. Even the event MPM only offloads *idle* keep-alive connections; an active slow request still ties up a thread. This is the structural reason nginx's fully event-driven, single-thread-per-many-connections model wins on C10k-style high-concurrency and slow-client workloads.

## Production Notes

**`.htaccess` is a performance footgun.** When `AllowOverride` is enabled, Apache walks every parent directory of every request looking for `.htaccess` files and re-reads them on each hit (no caching). On busy sites this is measurable overhead. Best practice: set `AllowOverride None` globally and move rules into the main config; only enable it where shared-hosting tenants genuinely need per-directory control.

**Pick the MPM deliberately.** A default install may still land on prefork, especially where `mod_php` is present. Prefork under real traffic consumes far more RAM than event. Moving PHP to **php-fpm** over `mod_proxy_fcgi` frees you to run the event MPM and is the standard modern layout; keeping `mod_php` locks you to prefork.

**Tuning knobs that actually matter:** `MaxRequestWorkers` (formerly `MaxClients`) caps concurrency — set too high and a traffic spike swaps the box to death; too low and requests queue. `KeepAliveTimeout` defaults are often too generous and pin workers on idle connections under the non-event MPMs. `Timeout` interacts with slow-client and slow-backend scenarios.

**TLS and HTTP/2.** `mod_ssl` (OpenSSL) handles TLS; HTTP/2 arrived via `mod_http2` in 2.4.17 (2015)[^4]. HTTP/2 with the prefork MPM is explicitly discouraged by the project — use event or worker. There is no HTTP/3 (QUIC) module in the mainline server as of the 2.4 line.

**Security posture.** A long history means a long CVE list; keep to a current 2.4.x patch release and subscribe to `announce@httpd.apache.org`. Common hardening: disable `mod_status`/`server-info` exposure, set `ServerTokens Prod` and `ServerSignature Off`, and audit loaded modules — a default install enables more than most sites use.

**Upgrades.** The 2.4 line has held ABI/config stability for over a decade, so patch upgrades are usually uneventful. The painful migration was 2.2 → 2.4: the authorization system was rewritten (`Order`/`Allow`/`Deny` replaced by `Require`), and many `.conf` files needed hand editing. There is no 2.6 or 3.0 mainline; 2.4 remains the supported series.

## When to Use / When Not

**Use when:**
- You're running classic PHP/CGI apps, cPanel/shared hosting, or anything that assumes `.htaccess`.
- You need per-directory config delegated to untrusted tenants without touching the main config.
- You want the widest module ecosystem and the most third-party documentation and Stack Overflow answers of any server.
- Compatibility and "it just works with everything" matter more than peak throughput.

**Avoid when:**
- You're building a high-concurrency reverse proxy / load balancer or serving many slow clients — nginx or a purpose-built proxy fits the workload better.
- You want automatic HTTPS and a near-zero config file — Caddy is dramatically simpler.
- You need HTTP/3, or a config format friendlier to version control and templating.
- You're optimizing RAM per connection at scale.

## Alternatives

- nginx/nginx — use instead when you need a high-concurrency reverse proxy, load balancer, or static-file server; event-driven model handles far more connections per byte of RAM.
- caddyserver/caddy — use instead when you want automatic Let's Encrypt TLS and a minimal config; single Go binary, sane defaults.
- traefik/traefik — use instead when routing is dynamic and container-native (Docker/Kubernetes) with service discovery rather than static vhosts.
- lighttpd/lighttpd — use instead for embedded or resource-constrained static serving where footprint dominates.
- apache/trafficserver — use instead as a dedicated caching forward/reverse proxy at CDN scale; a different Apache project, not the same codebase.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 1995-12 | First official release, atop patched NCSA httpd[^1]. |
| 1.3 | 1998-06 | Long-lived series; Windows support matured. |
| 2.0 | 2002-04 | MPM architecture and APR introduced; major rewrite. |
| 2.2 | 2005-12 | `mod_proxy` overhaul, smart filtering, caching improvements. |
| 2.4 | 2012-02 | event MPM as default, new `Require` authz, per-module logging[^3]. |
| 2.4.17 | 2015-10 | HTTP/2 support via `mod_http2`[^4]. |

## References

[^1]: Apache HTTP Server Project — About / project history. https://httpd.apache.org/ABOUT_APACHE.html
[^2]: Netcraft Web Server Survey — long-running tracking of Apache vs nginx market share. https://www.netcraft.com/blog/category/web-server-survey/
[^3]: Apache HTTP Server — "Overview of new features in Apache HTTP Server 2.4". https://httpd.apache.org/docs/2.4/new_features_2_4.html
[^4]: Apache HTTP Server — HTTP/2 guide (`mod_http2`, added in 2.4.17). https://httpd.apache.org/docs/2.4/howto/http2.html

## Tags

c, web-server, http, reverse-proxy, apache, tls, http2, infrastructure, self-hosted, cgi
