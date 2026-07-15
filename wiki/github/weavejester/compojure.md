# weavejester/compojure

> A concise routing DSL for Ring — the macro-based router that defined Clojure web development for a decade.

[GitHub repo](https://github.com/weavejester/compojure) ·
[API docs](http://weavejester.github.io/compojure) ·
[License: EPL-1.0](https://github.com/weavejester/compojure/blob/master/epl-v10.html)

## Overview

Compojure is a small library that adds a routing layer on top of [Ring][1], the Clojure HTTP server abstraction. It supplies a set of macros — `defroutes`, `GET`, `POST`, `context`, and friends — that let you map HTTP method + path patterns to Ring handler functions, with request destructuring built into the route form. It is deliberately narrow: it does routing and nothing else, delegating server adapters, middleware, and response handling to Ring itself.

Historically Compojure was one of the first Clojure web projects. It began around 2008–2009 as a broader micro-framework and was later stripped down to a routing library once Ring emerged as the community's server abstraction[^2]. For most of the 2010s it was the default answer to "how do I route HTTP in Clojure," and it remains widely deployed, including as the router that Luminus and many Leiningen templates scaffolded by default.

The defining tension is that Compojure's routes are **macros, not data**. A route is compiled into an opaque Ring handler function. That makes the DSL compact and readable, but it means you cannot introspect the route table, generate URLs by name, or reason about routes programmatically. This is precisely the limitation that motivated the data-driven routers (reitit, bidi) that have taken mindshare since roughly 2017. Compojure is stable and low-churn rather than actively evolving; the last release cadence has been slow but the project is maintained, not abandoned[^3].

## Getting Started

```clojure
;; deps.edn
compojure/compojure {:mvn/version "1.7.2"}
;; or Leiningen: [compojure "1.7.2"]
```

```clojure
(ns hello-world.core
  (:require [compojure.core :refer [defroutes GET POST context]]
            [compojure.route :as route]
            [ring.adapter.jetty :refer [run-jetty]]))

(defroutes app
  (GET "/" [] "<h1>Hello World</h1>")
  (GET "/user/:id" [id] (str "User " id))          ; path param destructured
  (POST "/echo" {body :body} (slurp body))          ; full request destructured
  (route/not-found "<h1>Page not found</h1>"))

(run-jetty app {:port 3000})
```

The binding vector after the path (`[id]`) destructures request parameters by name; a map form (`{body :body}`) destructures the raw Ring request. A route returning a string, map, or `nil` is coerced to a Ring response by Compojure's `render` protocol.

## Architecture / How It Works

Compojure sits between a Ring adapter and your handlers. The core pieces:

- **Route matching** is delegated to [Clout][4], a sibling library (also by weavejester) that compiles path patterns like `/user/:id` into regexes and returns the matched params. Compojure itself contains almost no matching logic.
- **`defroutes` / `routes`** combine multiple route handlers into one Ring handler. Each candidate route is tried in order; the first that returns a non-`nil` response wins, otherwise the combined handler returns `nil` (letting an outer handler or 404 take over). Routing is therefore **linear**, `O(n)` in the number of routes, driven by sequential `nil`-fallthrough.
- **The method macros** (`GET`, `POST`, `PUT`, `DELETE`, `ANY`, ...) expand into a `make-route` call that pairs a Clout matcher with a function body. The binding vector is macro-expanded into `let`-destructuring over the request's merged params.
- **`context`** prefixes a group of routes with a shared path segment and can bind path variables for the whole group. It nests by wrapping the inner routes' matcher.
- **`wrap-routes`** applies middleware to a route handler *after* matching, which is the idiomatic way to scope middleware to a subset of routes rather than the whole app.

Because routes are just Ring handler functions, they compose with ordinary function composition and with any Ring middleware. There is no separate "router" object or dispatch table — the entire route tree is a tree of closures produced at macro-expansion time.

## Production Notes

- **Routes are not introspectable.** There is no route table you can read, no named-route reverse routing (URL generation from a route name), and no built-in way to list all endpoints. Teams that need an OpenAPI spec, URL helpers, or route-based middleware analysis end up bolting on conventions or migrating to reitit. This is the single most common reason projects leave Compojure.
- **Linear matching cost.** With hundreds of routes the sequential `nil`-fallthrough matching is measurable, though rarely the bottleneck for typical apps. Data routers with trie/compiled dispatch scale better on very large route sets.
- **Destructuring foot-guns.** The `[id]` binding form pulls from the *merged* params map (route params + query + form), so you need `wrap-params` / `wrap-keyword-params` middleware in the stack for the expected keys to be present. Forgetting the middleware yields silent `nil` bindings rather than errors.
- **Return-value coercion is implicit.** A handler returning a bare string becomes a `200 text/html` response via the `render` protocol; returning `nil` means "no match, fall through," which occasionally surprises people whose handler legitimately wants to return an empty body (use an explicit response map).
- **Middleware ordering.** `wrap-routes` vs. wrapping the whole `defroutes` handler differ in whether middleware runs before or after route matching — a frequent source of confusion when scoping auth or content-negotiation middleware.
- **No async-first design.** Compojure predates Ring's async (3-arity) handler support; async works but the ergonomics are aimed at synchronous handlers.

## When to Use / When Not

**Use when:**
- You want a minimal, readable router and are comfortable with a macro DSL.
- You're maintaining or extending an existing Compojure/Luminus app.
- Your route set is small-to-medium and you don't need URL generation or route data.
- You value stability and a frozen API over new features.

**Avoid when:**
- You need routes as data — introspection, reverse routing, OpenAPI/Swagger generation, or per-route metadata (reach for reitit).
- You want request/response coercion and spec-driven validation at the route layer.
- You have a very large route table where compiled dispatch matters.
- You prefer explicit data structures over macros for testability.

## Alternatives

- metosin/reitit — data-driven router with fast dispatch, reverse routing, coercion, and OpenAPI; use instead when you need routes as data or spec validation.
- juxt/bidi — bidirectional routing (match and generate URLs) as pure data; use when reverse routing matters but you want minimalism.
- ring-clojure/ring — the layer beneath Compojure; use directly when your routing needs are trivial enough for a hand-written handler.
- pedestal/pedestal — fuller service framework with an interceptor model; use instead when you want an opinionated end-to-end stack rather than just a router.
- yogthos/luminus (template) — bundles Compojure or reitit into a batteries-included app; use when you want scaffolding rather than assembling Ring yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2008–2009 | Started as a broader Clojure web micro-framework[^2]. |
| — | ~2010 | Refactored to a routing-only library atop Ring[^2]. |
| 1.0.x | early 2010s | Stabilized routing DSL; Clout extracted for matching[^4]. |
| 1.6.x | 2017 | Maintenance era; ecosystem shift toward data routers begins. |
| 1.7.2 | 2025 | Current release; EPL, copyright James Reeves[^3]. |

## References

[^1]: Ring — Clojure HTTP server abstraction. https://github.com/ring-clojure/ring
[^2]: Compojure Wiki — project background and Getting Started. https://github.com/weavejester/compojure/wiki
[^3]: Compojure README and releases (v1.7.2, EPL, © 2025 James Reeves). https://github.com/weavejester/compojure
[^4]: Clout — the route-matching library used by Compojure. https://github.com/weavejester/clout

## Tags

clojure, routing, ring, http, web-framework, dsl, macros, backend, server, epl
