# GenieFramework/Genie.jl

> Full-stack MVC web framework for Julia — Rails-style productivity on a JIT-compiled scientific language.

[GitHub repo](https://github.com/GenieFramework/Genie.jl) ·
[Official website](https://genieframework.com) ·
[License: MIT](https://github.com/GenieFramework/Genie.jl/blob/main/LICENSE.md)

## Overview

Genie.jl is the dominant web framework in the Julia ecosystem — started in 2016 by Adrian Salceanu, 1.0 in July 2020[^1], now the backbone of the commercial Genie Framework stack (Genie.jl server + Stipple.jl reactive UI + SearchLight.jl ORM + Genie Builder VSCode plugin)[^2]. It deliberately imports the Rails/Django playbook: MVC scaffolding, file-system conventions, environments, migrations, a plugin system, and a REPL-first development loop.

The defining tension is its audience. Julia's users are overwhelmingly scientists and quants, not web developers — so Genie's real-world niche is not general web apps but *putting a web face on Julia compute*: dashboards, model-serving APIs, and interactive simulations that would otherwise require rewriting numerical code in Python or JS. The framework's evolution reflects this: the reactive low-code layer (Stipple, Vue.js-based) has become the flagship product surface, while classic server-rendered MVC development receives comparatively less attention.

At ~2.4k stars, 186 forks, and a push as recent as June 2026 (v6.0.0 shipped 2026-06-11[^3]), the project is actively maintained — but development is concentrated in a small team around the GenieFramework company, and 126 open issues against that team size means non-trivial bug-report latency. The company's revenue comes from Genie Builder and cloud offerings, which shapes where engineering effort goes.

## Getting Started

```julia
using Pkg; Pkg.add("Genie")
```

Micro-service style — routes in a script:

```julia
using Genie
using Genie.Renderer.Json

route("/hello") do
  "Welcome to Genie!"
end

route("/api/data") do
  (:series => rand(10)) |> json
end

up(8000, async = false)   # blocking server on :8000
```

Full MVC app scaffold:

```julia
using Genie
Genie.Generator.newapp_mvc("MyApp")   # creates app/, config/, db/, routes.jl
# then: cd MyApp; julia --project -e 'using Genie; Genie.loadapp(); up(async=false)'
```

## Architecture / How It Works

Genie is a conventions layer over **HTTP.jl** (the JuliaWeb HTTP server/client). Requests flow: HTTP.jl listener → `Genie.Router` (static, dynamic `:param`, and named routes across the usual HTTP verbs) → handler function → `Genie.Renderer`. Handlers are plain Julia closures; the router extracts path/query/POST params into a `Params` dict.

- **Templating** — `.jl.html` view files embed Julia expressions (`$var`, `<% %>` blocks) and are compiled to Julia functions, so subsequent renders are JIT-compiled code rather than string interpolation. HTML, JSON, Markdown, and JS renderers ship in the box[^1].
- **Channels** — WebSocket endpoints declared with `channel("/path") do ... end`, symmetric to `route()`. This is the transport Stipple builds on.
- **Stipple.jl** — the reactive layer: you declare a model with `@app` / reactive field macros, and Stipple keeps browser state (a Vue.js frontend) and the server-side Julia model in sync over WebSockets. Low-code in the sense that you write no JS; the tradeoff is that every bound variable round-trips through the server[^4].
- **SearchLight.jl** — a separate-package ORM (Postgres, MySQL, SQLite adapters) with migrations and model validations. Optional; Genie itself is DB-agnostic[^5].
- **Modularization** — v5 (2022) broke the former monolith apart: sessions, auth, caching, autoreload, and deployment helpers live in satellite packages (`GenieSession.jl`, `GenieAuthentication.jl`, etc.), keeping core Genie's dependency tree lean at the cost of assembly work for full-featured apps.

The coupling story: Genie core is honestly standalone, but the marketing and docs increasingly assume the full Genie Framework stack. Using Genie.jl purely as an HTTP/MVC layer is supported and common; using Stipple ties you to its Vue-based component model and the company's component libraries.

## Production Notes

**Time-to-first-response is the big one.** Julia JIT-compiles on first execution, so a freshly started Genie app can take tens of seconds before serving its first request, and each route's first hit pays compilation cost. Julia 1.9+ native code caching (pkgimages) improved this substantially; for serious deployments, bake a PackageCompiler.jl sysimage into your Docker image[^6]. This also makes naive serverless/autoscale deployment a poor fit — treat Genie apps as long-lived processes.

**Memory footprint.** A Julia runtime plus Genie typically idles at several hundred MB — fine for a dashboard VM, expensive for many small containers. Plan capacity accordingly.

**Concurrency.** HTTP.jl serves each request on a Julia task; CPU-bound handlers block others unless you start Julia with threads (`julia -t auto`) and write thread-aware code. Julia's GC and not-fully-mature multithreaded ecosystem mean the standard scale-out answer is multiple processes behind nginx/caddy.

**Stipple state is server-side and per-session.** Reactive apps hold each connected client's model in server memory and require the WebSocket to stay pinned to that process — horizontal scaling needs sticky sessions, and many concurrent users multiply memory usage. Genie's own cloud product exists partly because self-hosting this well is non-obvious.

**Upgrade pains.** v4 → v5 (2022) moved sessions/auth/caching into separate packages, breaking `Genie.Sessions`-style imports; community plugins lagged. v6.0.0 (June 2026) is a fresh major — check that satellite packages (GenieAuthentication, Stipple pins) have caught up before upgrading production apps[^3].

**Ecosystem thinness.** Compared to Rails/Django there is no deep middleware bazaar: authentication is a basic plugin, authorization another, and anything beyond (rate limiting, admin panels, background-job frameworks) you mostly build yourself from Julia packages. Documentation is split between genieframework.com and learn.genieframework.com, and OSS-path docs (classic MVC) are staler than the Genie Builder path.

## When to Use / When Not

**Use when:**
- You have substantial Julia compute (models, simulations, optimization) and want to expose it as a web app or API without a Python/JS rewrite.
- You're building internal dashboards or interactive scientific tools — Stipple's no-JS reactive model is genuinely fast to iterate for this.
- One language across data layer, logic, and (via Stipple) UI matters to your team.

**Avoid when:**
- The app is a conventional CRUD/content product with no Julia dependency — Rails, Django, Laravel, or Next.js have 100× the ecosystem and hiring pool.
- You need many small, fast-booting instances (serverless, scale-to-zero) — Julia's startup and memory profile fight you.
- You only need a REST API around Julia code — a micro-framework is less machinery.
- You require battle-tested auth/security middleware out of the box.

## Alternatives

- OxygenFramework/Oxygen.jl — FastAPI-style Julia micro-framework; use instead when you just need REST endpoints without MVC scaffolding.
- JuliaWeb/HTTP.jl — the server layer Genie sits on; use directly for maximum control and minimum dependencies.
- plotly/Dash.jl — use for pure data dashboards if you prefer the Dash/Plotly component model over Vue/Stipple.
- django/django — use when the product is a general web app and Julia interop can live behind an internal API.
- streamlit/streamlit — use for quick data-app UIs when the compute can be (or already is) Python.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2016-02 | Development starts; Rails-inspired MVC for Julia[^7]. |
| 1.0.0 | 2020-07-29 | First stable release[^1]. |
| 2.0.0 | 2021-06-28 | Rapid major-version cadence begins (v2–v4 within six months)[^3]. |
| 4.0.0 | 2021-11-01 | Last monolithic major[^3]. |
| 5.0.0 | 2022-07-22 | Modularization: sessions, auth, caching split into satellite packages[^3]. |
| 6.0.0 | 2026-06-11 | Current major line[^3]. |

## References

[^1]: Genie.jl README and release history. https://github.com/GenieFramework/Genie.jl
[^2]: Genie Framework component overview (Genie.jl + Stipple.jl + SearchLight.jl + Genie Builder). https://genieframework.com
[^3]: Genie.jl GitHub releases. https://github.com/GenieFramework/Genie.jl/releases
[^4]: Stipple.jl — reactive UI layer for Genie. https://github.com/GenieFramework/Stipple.jl
[^5]: SearchLight.jl — ORM for Julia. https://github.com/GenieFramework/SearchLight.jl
[^6]: PackageCompiler.jl — sysimages to cut Julia startup/JIT latency. https://github.com/JuliaLang/PackageCompiler.jl
[^7]: GitHub repo metadata: created 2016-02-19. https://api.github.com/repos/GenieFramework/Genie.jl

## Tags

julia, web-framework, mvc, full-stack, reactive-ui, low-code, dashboards, orm, websockets, data-apps
