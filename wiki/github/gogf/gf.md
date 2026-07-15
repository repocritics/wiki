# gogf/gf

> An all-in-one, modular application framework for Go — HTTP server, ORM, config, logging, and codegen in one dependency, with OpenTelemetry wired in.

[GitHub repo](https://github.com/gogf/gf) ·
[Official website](https://goframe.org) ·
[License: MIT](https://github.com/gogf/gf/blob/master/LICENSE)

## Overview

GoFrame (import path `github.com/gogf/gf/v2`) is a batteries-included
application framework for Go, first published in 2017[^1]. Where the mainstream
Go idiom is "standard library plus a thin router plus hand-picked libraries,"
GoFrame takes the opposite position: it ships an HTTP server, an ORM, a
configuration system, structured logging, caching, cron, validation, type
conversion, and a large set of container/utility packages under a single
versioned module. It is closest in spirit to Beego or Java's Spring Boot —
a framework you adopt wholesale, not a library you import piecemeal.

The project originated in and is still most heavily used by the
Chinese-speaking Go community, and its documentation was Chinese-first for
most of its life. An English site exists at goframe.org/en, but depth and
freshness still favor the Chinese docs — the single biggest practical barrier
for teams outside that ecosystem. At ~13k stars it is one of the larger Go
frameworks, though an order of magnitude behind Gin, and it is actively
maintained (releases and commits land regularly on `master`)[^2].

The defining tension is scope versus idiom. GoFrame gives you a coherent,
standardized project skeleton and code generation so that ten services in an
organization look the same — genuinely valuable at team scale. The cost is a
large surface area, heavy use of global singletons (the `g` convenience
package), and pervasive reflection in the conversion layer, all of which sit
uneasily with Go's preference for explicit, minimal wiring.

## Getting Started

```bash
go get -u github.com/gogf/gf/v2
# optional but central to the workflow: the gf CLI (scaffolding + codegen)
go install github.com/gogf/gf/cmd/gf/v2@latest
```

```go
package main

import (
	"github.com/gogf/gf/v2/frame/g"
	"github.com/gogf/gf/v2/net/ghttp"
)

func main() {
	s := g.Server()
	s.BindHandler("/hello", func(r *ghttp.Request) {
		r.Response.WriteJson(g.Map{"message": "hello", "name": r.Get("name").String()})
	})
	s.SetPort(8000)
	s.Run()
}
```

`g.Server()`, `g.DB()`, `g.Cfg()`, `g.Log()`, and `g.Redis()` return
process-wide singletons configured from a `config.yaml` discovered at startup —
the default programming model leans on these globals rather than explicit
construction.

## Architecture / How It Works

GoFrame is a set of independent packages unified by conventions and the `g`
facade. The major pieces:

- **`ghttp`** — the HTTP server and router. Supports both a handler-registration
  style and a struct-based "controller" style; the latter feeds an OpenAPI 3.0
  spec generated from request/response struct tags[^3].
- **ORM (`gdb`)** — a chainable query builder plus model layer with soft-delete,
  automatic created/updated timestamps, and driver packages for MySQL, PostgreSQL,
  SQLite, SQL Server, Oracle, and ClickHouse. It is not an entity-graph ORM like
  GORM; it stays closer to SQL.
- **`gf` CLI codegen** — the workflow centerpiece. `gf gen dao` reads a live
  database schema and generates DAO, `Entity`, and `DO` (data-object) types;
  `gf gen service` generates service interfaces from `logic` implementations;
  `gf gen ctrl` wires controllers from API definitions. The framework pushes a
  three-tier `controller → logic → dao` layout.
- **Observability** — OpenTelemetry tracing and metrics are integrated across
  the HTTP client/server, ORM, and logging, so context propagation is on by
  default rather than bolted on[^4].
- **Utilities** — `gconv` (universal type conversion), `gvar` (a dynamic
  variant type), `gvalid` (struct-tag validation), `gcfg`, `gcache`, `gcron`,
  and concurrent containers (`gmap`, `garray`, `gset`, `gpool`).

The `gconv`/`gvar` layer is what makes the API feel dynamic — request params,
config values, and DB columns all flow through reflection-based conversion. This
is the source of much of the convenience and, in hot paths, much of the cost.

## Production Notes

**Documentation language.** The most common real-world friction is not
technical: authoritative, up-to-date material is in Chinese. English docs cover
the basics but thin out for advanced ORM, codegen, and configuration topics.
Budget for reading Chinese docs (or machine translation) on non-trivial work.

**Globals and testability.** The idiomatic style routes everything through `g`
singletons reading a shared config file. This is fast to write but makes
dependency injection and test isolation more awkward than explicit constructor
wiring; teams that care about hermetic tests often wrap or avoid the `g` facade.

**Codegen is a commitment.** `gf gen dao` output is regenerated from the schema,
so the database is the source of truth and hand-edits to generated files are
overwritten. The three-tier structure pays off across many services but is heavy
for a single small binary.

**ORM surprises.** Soft-delete and auto-timestamp behavior are convention-driven
and can silently change query semantics (e.g. a `deleted_at IS NULL` predicate
injected automatically). Know the model conventions before trusting generated
SQL, and log built queries during development.

**Dependency weight.** Importing GoFrame pulls a broad tree; it is designed to
be the framework, not one library among many. Mixing it with an unrelated router
or ORM defeats the purpose and adds redundancy.

**Versioning.** Current work is the v2 line (`/v2` module path); v1 predates the
2021 rewrite and should not be used for new projects. Pin the minor version —
the framework moves and the utility packages occasionally adjust behavior.

## When to Use / When Not

**Use when:**
- You want one coherent, standardized skeleton across many services in an org.
- Database-driven code generation (DAO/DO/Entity, service interfaces) fits your
  workflow and you value structural consistency over minimalism.
- You want built-in OpenTelemetry, structured logging, config, and validation
  without assembling and version-matching them yourself.
- Your team reads Chinese docs comfortably or works in that ecosystem.

**Avoid when:**
- You prefer the minimalist Go idiom: a small router plus explicit, hand-picked
  libraries and no framework lock-in.
- The service is small enough that Gin/Echo plus `sqlc`/GORM is simpler.
- Your team needs deep English documentation for every advanced feature.
- Heavy reflection-based conversion in hot paths is a measured concern.

## Alternatives

- gin-gonic/gin — the default minimalist Go HTTP router; bring your own ORM,
  config, and logging. Use when you want maximum control and a small surface.
- labstack/echo — comparable minimalist router with a slightly larger built-in
  middleware set; use when you want Gin's model with more built-ins.
- beego/beego — the older all-in-one Chinese framework; the closest philosophical
  peer but less actively developed. Use only for legacy familiarity.
- go-kratos/kratos — Bilibili's microservice-first framework centered on gRPC and
  protobuf. Use when the shape of the system is RPC services, not HTTP apps.
- go-zero/go-zero — codegen-driven microservice framework with built-in
  resilience primitives. Use when you want opinionated microservices with API/RPC
  scaffolding.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-06 | Repository created; v1 line begins[^1]. |
| v2.0 | 2021 | Major rewrite; `/v2` module path, standardized error stack, OpenTelemetry integration[^2]. |
| v2.10.2 | 2026 | Current release line as referenced by the repository[^2]. |

## References

[^1]: gogf/gf repository — created 2017-06-29. https://github.com/gogf/gf
[^2]: GoFrame releases. https://github.com/gogf/gf/releases
[^3]: GoFrame docs, "API / OpenAPIv3". https://goframe.org/en
[^4]: GoFrame docs, "Observability / Tracing". https://goframe.org/en

## Tags

go, golang, web-framework, orm, microservices, opentelemetry, code-generation, backend, all-in-one, full-stack-framework
