# openresty/lua-nginx-module

> Embed LuaJIT into Nginx's event loop — script every request phase in Lua with
> non-blocking I/O, at near-C performance. The core of OpenResty.

[GitHub repo](https://github.com/openresty/lua-nginx-module) ·
[Official website](https://openresty.org/) ·
[License: BSD-2-Clause](https://github.com/openresty/lua-nginx-module#copyright-and-license)

## Overview

ngx_http_lua_module ("ngx_lua") embeds the LuaJIT VM into Nginx and exposes
hooks into every phase of HTTP request processing — rewrite, access, content,
header/body filters, logging, SSL handshake, upstream balancing. Started in
2010 by Xiaozhe Wang (chaoslawful) and Yichun "agentzh" Zhang, it is the
foundational component of OpenResty: the README itself says "if you are using
this module, then you are essentially using OpenResty"[^1]. It is the engine
underneath Kong, Apache APISIX, and a decade of CDN/WAF edge logic at
Cloudflare and elsewhere.

The defining trick is the **cosocket**: Lua code that looks synchronous
(`sock:connect()`, `sock:receive()`) but yields the current coroutine and
registers with Nginx's event loop, so a single worker multiplexes thousands of
concurrent requests without callbacks. Unlike Apache's mod_lua or Lighttpd's
mod_magnet, network I/O through the provided `ngx.*` API is fully
non-blocking[^1]. The tradeoff: any I/O *outside* that API (standard
`io.*`, `os.execute`, plain LuaSocket, blocking DNS via `socket.dns`) silently
blocks the entire worker — the module's biggest footgun.

Since v0.10.16 the standard PUC-Rio Lua interpreter is no longer supported;
LuaJIT (specifically OpenResty's maintained LuaJIT2 fork) is mandatory[^1].
The project is actively maintained (11.8k stars, ~2k forks, last push July
2026, latest release v0.10.29 in October 2025[^2]), but as a C module tracking
Nginx internals its release cadence is deliberate — roughly one or two
releases per year — and 392 open issues reflect a large, old surface area more
than neglect.

## Getting Started

The maintainers strongly discourage compiling this module into Nginx yourself
— the supported path is an OpenResty bundle, which ships a patched Nginx, the
optimized LuaJIT2 fork, and the mandatory `lua-resty-core` /
`lua-resty-lrucache` libraries[^3]:

```bash
# macOS
brew install openresty/brew/openresty
# Ubuntu: see https://openresty.org/en/linux-packages.html
```

```nginx
# nginx.conf
http {
    lua_package_path "/path/to/lua/?.lua;;";
    server {
        listen 8080;
        location = /hello {
            default_type text/plain;
            content_by_lua_block {
                ngx.say("Hello from ", ngx.var.remote_addr)
            }
        }
        location = /gate {
            access_by_lua_block {
                if ngx.var.remote_addr == "10.0.0.66" then
                    return ngx.exit(ngx.HTTP_FORBIDDEN)
                end
            }
            proxy_pass http://backend;
        }
    }
}
```

Manual builds require ngx_devel_kit (NDK), LuaJIT headers, and post-install of
lua-resty-core — `lua_load_resty_core off` is no longer allowed[^3]. Since
Nginx 1.9.11 it can also be built with `--add-dynamic-module` and loaded via
`load_module`.

## Architecture / How It Works

ngx_lua registers C handlers into Nginx's phase engine via configuration
directives (`rewrite_by_lua_block`, `access_by_lua_block`,
`content_by_lua_block`, `log_by_lua_block`, `init_worker_by_lua_block`,
`ssl_certificate_by_lua_block`, `balancer_by_lua_block`, and file variants).
Each worker process hosts **one shared LuaJIT VM**; per-request isolation is
achieved with lightweight Lua coroutines, not separate states[^1]. Loaded Lua
modules persist at worker level, which keeps memory small under load — and
means module-level state is shared across all requests in that worker.

When Lua code calls a cosocket or `ngx.sleep`, the C side yields the
coroutine, arms an Nginx event (epoll/kqueue), and resumes the coroutine when
the event fires. This is cooperative scheduling: CPU-bound Lua with no yield
points monopolizes the worker.

Two API implementations coexist: the original C-function bindings, and
`lua-resty-core`, an FFI-based reimplementation of the same `ngx.*` API that
LuaJIT can compile into machine code (C function calls abort JIT traces; FFI
calls do not). lua-resty-core has been mandatory since the v0.10.15/16
era[^3]. The module plugs into Nginx's `http` subsystem only; generic TCP/UDP
downstream scripting lives in the sibling stream-lua-nginx-module with a
compatible API[^1].

Cross-request/cross-worker state goes through `lua_shared_dict` — a
shared-memory LRU dictionary storing only serializable scalar types — or
external stores via cosockets. `ngx.location.capture` issues zero-copy
internal subrequests to other locations, though the resty-* client libraries
are recommended over subrequests to upstream modules[^1].

## Production Notes

- **Never block the event loop.** One blocking call (file I/O, `os.execute`,
  a non-cosocket library) stalls every request on that worker. Audit
  third-party Lua dependencies for hidden LuaSocket use.
- **Cosockets are not available in every phase**: `init_by_lua*`,
  `set_by_lua*`, `log_by_lua*`, `header_filter_by_lua*`, and
  `body_filter_by_lua*` cannot use them[^4]. The standard workaround is
  `ngx.timer.at(0, ...)` to detach work into a timer context — which then
  needs its own error handling and concurrency limits
  (`lua_max_pending_timers`).
- **Worker-shared VM state leaks.** Writing to Lua globals from request code
  is a classic bug: values persist across unrelated requests in the same
  worker but differ between workers. Use function-locals and `ngx.ctx` for
  per-request data; the module logs warnings for accidental globals.
- **Build from the OpenResty bundle.** Stock LuaJIT and unpatched
  Nginx/OpenSSL disable or slow down features (e.g., SSL-phase hooks need
  OpenResty's OpenSSL patches); the maintainers state this explicitly[^3].
- **`lua_code_cache off` is development-only** — it reloads Lua on every
  request and destroys throughput.
- **Subrequest limits**: `ngx.location.capture` buffers the entire subrequest
  response in memory and predates HTTP/2 — plan around resty-* clients for
  upstream traffic.
- **Memory ceiling**: stock LuaJIT x64 limits GC-managed memory to ~2 GB per
  VM; the OpenResty LuaJIT2 fork's GC64 mode lifts this. Another reason not
  to use vanilla LuaJIT.
- Upgrades track OpenResty releases, which can lag Nginx mainline by months;
  the current release supports Nginx cores from 1.6.x through 1.31.x[^2].

## When to Use / When Not

**Use when:**
- You need programmable edge logic — auth, rate limiting, routing, header
  rewriting, WAF rules — inside Nginx without proxy hops to an app server.
- You are building or extending an API gateway (Kong and APISIX are
  precedents) and want C-adjacent performance with scripting velocity.
- Your team can commit to the `ngx.*` cosocket discipline and the OpenResty
  library ecosystem (lua-resty-redis, lua-resty-mysql, etc.).

**Avoid when:**
- You want a general web application framework — the programming model is
  phase handlers in a server config, not MVC; ecosystem breadth is far below
  Node/Python/Go.
- Your logic is CPU-heavy: one shared VM per worker and cooperative
  scheduling make sustained computation a latency hazard.
- You need first-party Nginx support — F5's official scripting answer is
  njs, and this module requires a custom or OpenResty build.
- Lua is a hiring/maintenance liability for your organization.

## Alternatives

- nginx/njs — the official Nginx JavaScript scripting engine; use it when you
  want first-party support on stock Nginx packages and lighter scripting needs.
- apache/apisix — full API gateway built on top of this module; use it when
  you want gateway features (plugins, dashboard, etcd config) rather than raw
  phase scripting.
- Kong/kong — same story with a plugin SDK and enterprise ecosystem around
  OpenResty.
- envoyproxy/envoy — Lua filters and WASM extensions in a modern proxy; use
  it in service-mesh/xDS environments instead of Nginx.
- haproxy/haproxy — embedded Lua actions/services; use it when your base is
  HAProxy load balancing rather than an HTTP content server.

## History

| Version | Date | Notes |
|---------|------|-------|
| early 0.1.x | 2010 | Project started by chaoslawful and agentzh on ngx_devel_kit[^5]. |
| v0.10.16 | ~2020 | Standard PUC-Rio Lua support dropped; LuaJIT-only[^1]. |
| v0.10.2x | 2021–2024 | Incremental releases tracking new Nginx cores (1.21–1.27). |
| v0.10.29 | 2025-10-24 | Current release; Nginx up to 1.31.x supported[^2]. |

## References

[^1]: lua-nginx-module README — Description, Data Sharing, and stream module notes. https://github.com/openresty/lua-nginx-module#description
[^2]: lua-nginx-module README — Version (v0.10.29, 2025-10-24) and Nginx Compatibility. https://github.com/openresty/lua-nginx-module#version
[^3]: lua-nginx-module README — Installation (OpenResty bundle recommendation, lua-resty-core requirement). https://github.com/openresty/lua-nginx-module#installation
[^4]: lua-nginx-module README — Known Issues, "Cosockets Not Available Everywhere". https://github.com/openresty/lua-nginx-module#cosockets-not-available-everywhere
[^5]: GitHub repository metadata: created 2010-04-16; copyright notices credit Xiaozhe Wang (2009–2017) and Yichun Zhang / OpenResty Inc. (2009–2025). https://github.com/openresty/lua-nginx-module#copyright-and-license

## Tags

c, lua, luajit, nginx, openresty, web-server, api-gateway, reverse-proxy, embedded-scripting, non-blocking-io, networking
