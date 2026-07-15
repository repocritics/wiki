# gobuffalo/buffalo

> A Rails-style, batteries-included web development ecosystem for Go — routing, ORM, templating, migrations, and an asset pipeline wired together behind one CLI.

[GitHub repo](https://github.com/gobuffalo/buffalo) ·
[Official website](http://gobuffalo.io) ·
[License: MIT](https://github.com/gobuffalo/buffalo/blob/main/LICENSE)

## Overview

Buffalo is a full-stack web framework for Go, created by Mark Bates and first
released around 2016[^1]. Its explicit model is Ruby on Rails: rather than being
a minimal HTTP library, it scaffolds an entire project — routing, an ORM with
migrations, server-rendered templates, sessions, and a front-end asset pipeline —
and glues them together behind a `buffalo` CLI with Rails-like generators. The
pitch is that `buffalo new` yields a working web app skeleton instead of an
afternoon spent assembling one library at a time.

The defining tension is that this convention-heavy, "holistic environment"
philosophy runs against the grain of the Go community, which favours small,
composable, standard-library-adjacent packages. Buffalo is itself thin glue — it
wraps `gorilla/mux` for routing, `gobuffalo/plush` for templates, and
`gobuffalo/pop` for the ORM[^2] — but presents them as one opinionated stack with
a large surface of `gobuffalo/*` support packages. That opinionation is the value
for teams arriving from Rails, and the friction for Go developers who would rather
pick their own router and write SQL.

Buffalo saw its widest adoption in roughly 2017–2019; cadence has since cooled.
The `main` branch was last pushed in early 2026, the `v1` branch is the current
stable line[^3], and the project actively supports only the last two Go releases.
It remains usable and maintained, but is no longer fast-moving — a maintenance
posture that should factor into any adoption decision.

## Getting Started

Install the CLI, then scaffold and run a project:

```bash
go install github.com/gobuffalo/cli/cmd/buffalo@latest

buffalo new coke        # scaffolds a full app (DB, routing, assets)
cd coke
buffalo dev             # hot-reloading dev server on :3000
```

A minimal handler and route:

```go
// actions/app.go
func App() *buffalo.App {
    app := buffalo.New(buffalo.Options{Env: envy.Get("GO_ENV", "development")})
    app.GET("/", HomeHandler)
    return app
}

// actions/home.go
func HomeHandler(c buffalo.Context) error {
    // renders templates/home/index.plush.html, or JSON/XML by content type
    return c.Render(http.StatusOK, r.HTML("home/index.plush.html"))
}
```

Database work goes through the `pop`/`soda` layer, driven from the CLI:

```bash
buffalo pop create -a          # create all databases from database.yml
buffalo pop generate migration add_users
buffalo pop migrate            # apply migrations
```

## Architecture / How It Works

Buffalo is best understood as an orchestration layer over a fixed set of
libraries plus a code-generating CLI:

- **Routing** — `gorilla/mux` under a `buffalo.App`, chosen for flexibility over
  raw speed[^2]. Middleware is registered per-app or per-group; groups map cleanly
  to resource controllers.
- **Templating** — `gobuffalo/plush`, an ERB-like engine picked over the standard
  `html/template` for a more expressive, Rails-familiar syntax[^2]. Templates are
  `*.plush.html`.
- **ORM / migrations** — `gobuffalo/pop` and its `soda` CLI (surfaced as
  `buffalo pop`) provide models, `fizz` DSL migrations, and support for Postgres,
  MySQL, SQLite, and CockroachDB[^2]. Pop is a separate project with its own
  learning curve and can be used entirely outside Buffalo.
- **Sessions / cookies / websockets** — the broader Gorilla toolkit, keeping
  Buffalo's own core small.
- **Generators & tasks** — `buffalo generate resource`, `buffalo generate action`,
  and Grift tasks (`buffalo task`) reproduce the Rails generator experience.

The two operational commands are `buffalo dev` and `buffalo build`. `dev` watches
the source tree and rebuilds on change. `build` compiles the app — templates,
migrations, and static assets included — into a single deployable binary. Asset
and template embedding historically relied on `gobuffalo/packr` (v1, then v2);
after Go 1.16 the ecosystem moved toward the standard `//go:embed` directive[^4],
which removed a long-standing class of build problems.

The most architecturally consequential decision is the **front-end asset
pipeline**: a default `buffalo new` app scaffolds a webpack/Node.js toolchain for
JavaScript and SCSS. This means a Go project pulls in `npm` and `node_modules`,
a coupling many Go developers found surprising. Later versions let you opt out
(`buffalo new --skip-webpack` / API-only layouts) to get a pure-Go project.

## Production Notes

**The Node/webpack coupling is the classic footgun.** Unless you scaffold with
`--skip-webpack` or an API layout, your Go build now also depends on a working
Node toolchain, and CI images must carry both. Teams that want plain Go should
choose the asset-free layout from the start; retrofitting later is more work.

**Asset embedding history matters on upgrades.** Projects generated in the packr
v1/v2 era carry `packr` imports and generated `*-packr.go` files. Migrating those
to `//go:embed` is a common and non-trivial upgrade step; mixed setups produce
confusing "asset not found in binary" failures at runtime rather than build time.

**Pop is a distinct system to learn.** The ORM is not "Buffalo" — it is a separate
project with its own conventions (`fizz` migrations, `pop.Connection`,
`database.yml`, eager-loading semantics), documented in the pop repo rather than
the Buffalo docs, which trips up newcomers.

**Dependency sprawl and momentum.** A Buffalo app transitively pulls in many small
`gobuffalo/*` modules (envy, flect, plush, pop, packr, grift, and more),
concentrating supply-chain and maintenance risk in one org whose activity has
slowed since the 2018–2019 peak. With low open-issue counts and infrequent pushes,
response times on bugs and Go-version compatibility can lag; Buffalo supports only
the latest two Go releases, so staying on an old Go is not a supported path.
Evaluate whether it will keep pace with your Go upgrade cadence before committing a
long-lived service to it.

**GOPATH is not supported.** Buffalo requires Go modules; running under GOPATH mode
breaks large parts of the ecosystem[^5].

## When to Use / When Not

**Use when:**
- Your team is coming from Rails/Django and wants the same conventions, generators,
  and full-stack structure in Go.
- You are building a server-rendered, database-backed web app and want migrations,
  templating, and routing decided for you.
- You value a single scaffolded project layout over assembling your own stack.

**Avoid when:**
- You are building an API-only or microservice backend — a router like chi/echo/gin
  plus your own ORM is lighter and more idiomatic.
- You want minimal dependencies and dislike a Node/webpack toolchain in a Go repo.
- You need a framework with high maintenance velocity and rapid Go-version support.
- You prefer to choose each component (router, DB layer, templates) independently.

## Alternatives

- gin-gonic/gin — fast, minimal HTTP framework; API-first, bring your own ORM and
  templates. Use when you want speed and control over a full-stack scaffold.
- labstack/echo — similar minimal router/middleware framework; use for services
  where Buffalo's batteries are dead weight.
- go-chi/chi — small idiomatic `net/http` router; use when you want to compose your
  own stack the Go-community way rather than adopt an opinionated one.
- beego/beego — the other Rails-style full-stack Go framework; use when you want
  batteries-included but prefer beego's MVC and tooling.
- gofiber/fiber — Express-style framework on fasthttp; use when raw throughput
  matters more than net/http compatibility.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016 | Public release; Rails-inspired full-stack Go framework by Mark Bates[^1]. |
| 0.11 | 2018 | "Road to 1.0" begins; Go modules required, GOPATH deprecated[^5]. |
| 0.16 | 2020 | Asset pipeline / packr and generator changes during modules transition. |
| 0.18 | 2021–2022 | Late 0.x line stabilising toward 1.0; go:embed adoption in ecosystem[^4]. |
| 1.0 | 2023 | First stable major release; `v1` becomes the supported branch[^3]. |

## References

[^1]: Buffalo documentation and project home. http://gobuffalo.io
[^2]: Buffalo README, "Shoulders of Giants" — dependencies on gorilla/mux (routing), gobuffalo/plush (templating), and gobuffalo/pop (ORM). https://github.com/gobuffalo/buffalo
[^3]: Buffalo README, "Versions" — `v1` is the current stable branch, `main` is mainstream development. https://github.com/gobuffalo/buffalo
[^4]: Go 1.16 `embed` package, which the Buffalo/packr ecosystem migrated toward for asset embedding. https://pkg.go.dev/embed
[^5]: "The road to 1.0: requiring modules" — Buffalo blog on dropping GOPATH support. https://blog.gobuffalo.io/the-road-to-1-0-requiring-modules-5672c6b015e5

## Tags

go, golang, web-framework, full-stack, rails-like, orm, mvc, server-rendered, batteries-included, gorilla-mux
