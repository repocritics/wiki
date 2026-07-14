# gorilla/mux

> An HTTP request router and URL matcher for Go, matching requests to handlers by path, host, method, header, and query — with reversible ("buildable") named routes.

[GitHub repo](https://github.com/gorilla/mux) ·
[Official website](https://gorilla.github.io) ·
[License: BSD-3-Clause](https://github.com/gorilla/mux/blob/main/LICENSE)

## Overview

`gorilla/mux` is the router from the Gorilla web toolkit, one of the oldest and most widely used pieces of Go web infrastructure — the repository dates to 2012[^1]. Its name is short for "HTTP request multiplexer." A `mux.Router` implements the standard `http.Handler` interface, so it drops in anywhere the standard library's `http.ServeMux` would go, but adds path variables (`/users/{id}`), optional per-segment regular expressions, and matching on host, method, scheme, header, and query values. Its distinguishing feature is **reversible routing**: a named route can generate its own URL from a set of variable values, so links and redirects stay in sync with route definitions.

The library's history is unusual and matters for anyone adopting it in 2026. In December 2022 the entire Gorilla toolkit — including mux — was **archived** by its maintainer, who stepped away[^2]. After community concern, the project was handed to a new maintainer team and un-archived in early 2023[^3]. It is active again but effectively feature-complete and in maintenance mode: the last substantive push was in 2024, and development is conservative by design.

The other context that matters is that Go 1.22 (February 2024) added method-based and wildcard pattern matching to the standard library's `http.ServeMux`[^4], closing much of the gap that historically made a third-party router necessary. mux still does more (regex segments, host matching, URL reversing, subrouter grouping), but for many new services the standard library is now enough, and mux's role has shifted from "default choice" to "reach for it when you need its specific features."

## Getting Started

```sh
go get -u github.com/gorilla/mux
```

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/gorilla/mux"
)

func main() {
	r := mux.NewRouter()
	r.HandleFunc("/articles/{category}/{id:[0-9]+}", articleHandler).
		Methods("GET").
		Name("article")
	http.ListenAndServe(":8000", r)
}

func articleHandler(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r) // route variables live on the request context
	fmt.Fprintf(w, "category=%s id=%s\n", vars["category"], vars["id"])
}
```

Reverse a named route to build a URL — the payoff for naming routes:

```go
u, _ := r.Get("article").URL("category", "tech", "id", "42")
// u.Path == "/articles/tech/42"
```

## Architecture / How It Works

A `Router` holds an ordered slice of `*Route` values. On each request it walks that slice **in registration order** and returns the first route whose matchers all pass — a linear O(n) scan, not a tree or trie[^5]. This is the single most important fact about mux's behavior and performance. Because matching is ordered, a broad route registered early (e.g. `PathPrefix("/")`) will shadow anything after it, and the fix is ordering, not priority.

Each `Route` is a chain of matchers built by the fluent API: `Path`, `PathPrefix`, `Host`, `Methods`, `Schemes`, `Headers`, `Queries`, and arbitrary `MatcherFunc`. Path and host templates with `{name}` or `{name:regexp}` segments are compiled to a Go `regexp` at registration time; an unqualified `{name}` matches everything up to the next slash. Matched variables are stored on the request's `context.Context` and read back with `mux.Vars(r)`. (Before Go 1.7 added `*http.Request.Context()`, this used the separate `gorilla/context` package; that dependency is gone in modern versions.)

**Subrouters** are the main structural tool. `r.Host("x.com").Subrouter()` or `r.PathPrefix("/api").Subrouter()` returns a nested router whose routes are only tested if the parent matcher passes, which both groups shared conditions and prunes the linear scan. **URL reversing** works because each named route retains its template and can substitute variables back in via `URL()`, `URLHost()`, or `URLPath()`, validating each value against its segment regex so a generated URL is guaranteed to match the route that produced it.

## Production Notes

- **Linear matching scales with route count.** Matching cost grows with the number of registered routes and the complexity of their regexes. For dozens or a few hundred routes this is irrelevant. For thousands of routes on a hot path, a radix-tree router (chi, httprouter) will measurably outperform mux. Subrouters mitigate this by short-circuiting whole groups.
- **`StrictSlash` is a footgun.** `r.StrictSlash(true)` makes `/path` and `/path/` redirect to each other with a 301. On non-GET requests a 301 can cause the client to replay the request as a GET and silently drop the body[^6]. Prefer registering explicit paths or handling trailing slashes yourself for APIs that take POST/PUT bodies.
- **Path cleaning.** By default mux cleans paths (collapsing `//` and `..`) and 301-redirects to the clean form. `r.SkipClean(true)` disables this when you need to pass raw, possibly-encoded paths through (e.g. a proxy). Know which behavior you want before you ship.
- **`mux.Vars` on an unmatched request returns an empty map, not nil-safe magic.** Reading a variable that isn't in the route yields the zero value (`""`), so a typo in a variable name fails silently rather than erroring.
- **`CORSMethodMiddleware` is narrow.** It only sets `Access-Control-Allow-Methods` from a route's registered methods; you still need your own handling for `Access-Control-Allow-Origin` and preflight. It is not a full CORS solution.
- **Maintenance posture.** The project is stable and still receives fixes, but it is not adding features and was briefly archived. Treat it as mature-and-frozen. That is fine — and arguably a virtue — for existing services; it is a reason to weigh alternatives for greenfield work.

## When to Use / When Not

**Use when:**
- You need regex path/host segments, host-based matching, or query/header matching that the standard library's patterns don't cover.
- You rely on **URL reversing** from named routes to keep links and redirects consistent.
- You have an existing Gorilla-based codebase — mux is stable and there is no urgency to migrate.
- Your route count is modest and clarity matters more than nanoseconds per match.

**Avoid when:**
- You're starting fresh on Go 1.22+ and your routing needs (method + path wildcards) fit the standard `http.ServeMux` — fewer dependencies, no maintenance question.
- You have thousands of routes or extreme per-request latency budgets — a radix-tree router is faster.
- You want an actively evolving router with a large middleware ecosystem — chi is the more common modern default.

## Alternatives

- go-chi/chi — idiomatic, radix-tree router with a composable middleware ecosystem and `http.Handler` compatibility; use instead when you want mux-like ergonomics with better performance and active development.
- julienschmidt/httprouter — very fast radix-tree router; use when raw routing throughput dominates and you can accept its stricter, no-regex, no-route-conflict model.
- gin-gonic/gin — full web framework (router plus binding, rendering, middleware); use when you want batteries included rather than just a router.
- labstack/echo — comparable full framework with its own router and middleware stack; use when you want an all-in-one alternative to gin.
- golang/go (standard library `net/http.ServeMux`, 1.22+) — method and wildcard patterns with zero dependencies; use when your routing is simple and you want nothing to maintain.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2012-10 | Router extracted into the Gorilla web toolkit[^1]. |
| context on request | 2016 | Route vars move to `net/http` request context (Go 1.7), dropping `gorilla/context`. |
| v1.8.0 | 2020-08 | Route walking, `GetVarNames`, ongoing matcher refinements. |
| v1.8.1 | 2022-09 | Maintenance release. |
| archived | 2022-12 | Gorilla toolkit archived by its maintainer[^2]. |
| revived | 2023 | New maintainer team un-archives and resumes the toolkit[^3]. |

## References

[^1]: gorilla/mux repository, created 2012-10-02. https://github.com/gorilla/mux
[^2]: The Register, "Gorilla web toolkit gets archived" — coverage of the December 2022 archival. https://www.theregister.com/2022/12/12/gorilla_go_project/
[^3]: gorilla/mux issue thread and README on the toolkit's revival under new maintainers (2023). https://github.com/gorilla/mux
[^4]: Go 1.22 release notes, enhanced `net/http.ServeMux` routing patterns. https://go.dev/doc/go1.22#enhanced_routing_patterns
[^5]: gorilla/mux `mux.go` / `route.go` — ordered slice of routes matched sequentially. https://github.com/gorilla/mux/blob/main/mux.go
[^6]: gorilla/mux documentation on `StrictSlash` redirect behavior. https://pkg.go.dev/github.com/gorilla/mux#Router.StrictSlash

## Tags

go, golang, http-router, url-routing, web, middleware, request-multiplexer, gorilla-toolkit, url-reversing, maintenance-mode
