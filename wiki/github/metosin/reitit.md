# metosin/reitit

> A data-driven router for Clojure/ClojureScript, where routes are plain data and coercion, middleware, and API docs fall out of that data.

[GitHub repo](https://github.com/metosin/reitit) ·
[Documentation](https://cljdoc.org/d/metosin/reitit/) ·
[License: EPL-1.0](https://github.com/metosin/reitit/blob/master/LICENSE)

## Overview

reitit is a routing library for the Clojure and ClojureScript ecosystems, maintained by the Finnish consultancy Metosin since 2017[^1]. Its central idea is that a route table is ordinary Clojure data — a nested vector of path strings paired with maps — and that everything else (route matching, reverse routing, request/response coercion, middleware chains, Swagger/OpenAPI generation) is a transformation over that data. This is the same "data-driven" philosophy Metosin pursues across its other libraries (malli, muuntaja, sieppari), and reitit is the routing member of that family.

The library spans server and browser. On the server it plugs into Ring, into an interceptor-based HTTP stack (Pedestal or Sieppari), and generates API documentation from coercion schemas. In the browser (`reitit-frontend`) it handles HTML5 history routing for single-page apps. The same core router and route-data model back both, which is unusual — most Clojure routing libraries pick a side.

The defining tension is expressiveness versus indirection. Because routes are data and behavior is assembled from that data at router-creation time, a reitit setup can be hard to read top-to-bottom: what a route actually *does* depends on inherited route data, the coercion implementation you chose, and the order of a middleware/interceptor vector defined elsewhere. In exchange you get fast matching, real reverse routing, conflict detection, and generated docs that most alternatives (notably Compojure) never offered. Metosin classifies the project as "stable" in its own lifecycle model[^2].

## Getting Started

Add the bundled dependency (deps.edn or Leiningen). reitit targets Clojure 1.11+ and Java 11+, and is tested against LTS releases 11, 17, 21, and 25[^3].

```clj
;; deps.edn
{:deps {metosin/reitit {:mvn/version "0.10.1"}}}
```

```clj
(require '[reitit.core :as r])

(def router
  (r/router
    [["/api/ping" ::ping]
     ["/api/orders/:id" ::order]]))

(r/match-by-path router "/api/orders/2")
;; #Match{:template "/api/orders/:id"
;;        :data {:name ::order}
;;        :path-params {:id "2"}
;;        :path "/api/orders/2"}

(r/match-by-name router ::order {:id 2})
;; #Match{:template "/api/orders/:id" :path "/api/orders/2" ...}
```

The `match-by-name` call is bi-directional routing: a route can be resolved back to a URL from its name and params, which is how you build links without hardcoding paths.

## Architecture / How It Works

reitit is layered, and the layers ship as separate artifacts you compose:

- **`reitit-core`** — the router itself. `r/router` takes route data and returns an implementation chosen by the shape of the routes. Static routes go into a lookup map; routes with path parameters are compiled into a trie or a linear/segment matcher. This compile-once-at-startup step is why matching is fast and why route conflicts are detected eagerly rather than at request time.
- **`reitit-ring`** — adapts the core router to the Ring handler model, adds per-method routing (`:get`/`:post` maps), a default handler, and middleware composition where middleware can be expressed as data and reordered/deduplicated.
- **`reitit-http`** — the same idea against an interceptor model instead of Ring middleware, executed by Sieppari (`reitit-sieppari`) or Pedestal (`reitit-pedestal`).
- **Coercion** (`reitit-malli`, `reitit-spec`, `reitit-schema`) — a pluggable protocol. You declare `:parameters` and `:responses` schemas in route data; coercion middleware/interceptors validate and transform request and response bodies, and the *same* schemas feed doc generation.
- **API docs** (`reitit-swagger`, `fi.metosin/reitit-openapi`, `reitit-swagger-ui`) — walk the compiled routes and their coercion schemas to emit a Swagger 2 or OpenAPI 3 spec. No separate annotation layer; the docs are a projection of the route data.

The most important architectural concept is **route data inheritance**. Route data attached to a parent path segment (or to the router's top-level `:data`) is merged down into child routes. Middleware, coercion, and content negotiation are typically declared once at the router level and inherited by every route, then overridden locally where needed. This is what makes large route tables terse, and also what makes them hard to trace — the effective configuration of any single endpoint is the merge of several layers you have to reconstruct mentally.

Note the group-id split: newer `reitit-openapi` publishes under the verified `fi.metosin` Maven group, while the older modules stay under `metosin` for compatibility[^1]. Mixing the two group-ids in one project is expected, not a mistake.

## Production Notes

- **Startup cost, not per-request cost.** Router construction compiles routes and resolves the middleware/interceptor chain. For big route tables this is measurable work that happens once at boot. Recreating the router on every request (easy to do accidentally during REPL-driven development or with a badly-placed `ring-handler`) throws away the entire performance model — build the router once and reuse it.
- **Middleware/interceptor ordering is load-bearing and implicit.** The order of the `:middleware` vector is the execution order, and coercion, exception handling, and content negotiation must be arranged correctly relative to each other. A misordered chain fails in ways that look like coercion bugs. This is the most common source of "why is my 400 not being formatted" support questions.
- **Coercion is opt-in per stack.** `reitit-core` and `reitit-ring` do not coerce anything until you add a coercion implementation and the coercion middleware. It is entirely possible to declare `:parameters` schemas that are silently ignored because the middleware was never wired in — the schemas are just data otherwise.
- **Error message quality depends on the coercion library.** malli generally produces the friendliest errors and is Metosin's own recommendation; spec errors are the rawest. The choice is not purely stylistic — it affects what your API returns to clients on validation failure.
- **`match-by-path` returns path params as strings** unless coercion is applied; the `:id "2"` versus `:id 2` distinction in reverse vs. forward routing trips people up.
- **Upgrade friction is generally low.** reitit has stayed on 0.x for its whole life with a stable core API; most breakage over the years has come from the coercion/schema libraries underneath (malli, spec-tools) rather than reitit itself.

## When to Use / When Not

**Use when:**
- You want validated, self-documenting HTTP APIs in Clojure with Swagger/OpenAPI generated from the same schemas you validate against.
- You need real reverse routing, route conflict detection, or a single routing model shared between a Ring backend and a ClojureScript SPA.
- You are already in the Metosin data-driven stack (malli, muuntaja) and want routing that fits it.

**Avoid when:**
- You want a tiny, obvious router you can read top to bottom — Compojure's macro-and-forms style is more transparent for small apps, at the cost of no reverse routing or docs.
- Your team is unwilling to invest in understanding route-data inheritance and middleware ordering; the indirection outweighs the payoff for trivial services.
- You are not on Clojure/Script at all — reitit is JVM/JS-Clojure only.

## Alternatives

- weavejester/compojure — the long-standing Ring routing DSL; use it when you want minimal, macro-based routing and don't need reverse routing, coercion, or generated docs.
- juxt/bidi — data-driven bidirectional routing that predates reitit; use it when you want reverse routing in plain data but not the coercion/middleware/OpenAPI machinery.
- pedestal/pedestal — a full interceptor-based service framework with its own router; use it when you want the whole Pedestal model rather than a routing library (reitit can also run on Pedestal's interceptors).
- metosin/compojure-api — Metosin's earlier coercion+Swagger stack built on Compojure; effectively superseded by reitit for new projects, but still seen in legacy codebases.
- ring-clojure/ring — the underlying HTTP abstraction, not a router; reitit sits on top of it.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017 | First public release by Metosin; data-driven core router[^1]. |
| 0.2.0 | 2019 | Broadened Ring/coercion tooling; documented in Metosin's 0.2.0 post[^4]. |
| 0.3.0 | 2019 | "Faster and friendlier" — matcher and error-message improvements[^5]. |
| 0.5.x | ~2020–2021 | malli coercion matures as the recommended path. |
| 0.7.x | ~2024 | OpenAPI 3 support via new `fi.metosin/reitit-openapi` group[^1]. |
| 0.10.1 | 2026 | Current release; Clojure 1.11 / Java 11 baseline, tested to Java 25[^3]. |

## References

[^1]: reitit README — module list, `fi.metosin` group-id note, and project description. https://github.com/metosin/reitit
[^2]: Metosin open-source project lifecycle model ("stable" status). https://github.com/metosin/open-source#project-lifecycle-model
[^3]: reitit README — "Reitit requires Clojure 1.11 and Java 11 … tested with the LTS releases Java 11, 17, 21 and 25." https://github.com/metosin/reitit
[^4]: Metosin blog, "Welcome Reitit 0.2.0!" https://www.metosin.fi/blog/reitit020/
[^5]: Metosin blog, "Faster and Friendlier Routing with Reitit 0.3.0." https://www.metosin.fi/blog/faster-and-friendlier-routing-with-reitit030/

## Tags

clojure, clojurescript, routing, ring, data-driven, http, api, coercion, openapi, swagger, middleware, interceptors
