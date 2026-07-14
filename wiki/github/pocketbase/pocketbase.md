# pocketbase/pocketbase

> Open-source backend-as-a-service in a single Go binary: embedded SQLite, realtime subscriptions, auth, file storage, and an admin UI.

[GitHub repo](https://github.com/pocketbase/pocketbase) ·
[Official website](https://pocketbase.io) ·
[License: MIT](https://github.com/pocketbase/pocketbase/blob/master/LICENSE.md)

## Overview

PocketBase is a self-hosted backend that bundles a database, REST-ish API, authentication, file storage, realtime subscriptions, and an admin dashboard into one executable. It was first published in July 2022 by Gani Georgiev and remains a predominantly single-maintainer project[^1]. The pitch is concrete: download a binary, run `./pocketbase serve`, and you have a working backend with a web admin UI and generated APIs — no Docker, no external database, no service mesh.

The defining architectural choice is **embedded SQLite as the only database**. This is what makes the "one file" story true — there is no separate database process to run, back up, or connect to. It is also the project's hard ceiling: SQLite is single-writer and single-node, so PocketBase scales vertically (bigger box) but not horizontally (no built-in clustering or multi-primary replication). For the solo developer, indie SaaS, prototype, or internal tool that this targets, that tradeoff is usually a feature. For anything expecting high concurrent write throughput or multi-region deployment, it is a wall.

The second thing to internalize: **PocketBase is pre-1.0 and does not guarantee backward compatibility**[^2]. The README says so explicitly. Minor version bumps have shipped breaking changes to both the data layer and the Go/JS extension APIs. The v0.23 release (late 2024) was a large, backward-incompatible refactor of the core Go API and event hooks[^3]. Treat every `v0.x` upgrade as a migration, not a patch.

## Getting Started

**As a standalone app** — download a prebuilt executable from the [releases page](https://github.com/pocketbase/pocketbase/releases), extract, and run:

```sh
./pocketbase serve
# admin UI at http://127.0.0.1:8090/_/
# REST API at http://127.0.0.1:8090/api/
```

**As a Go framework** — embed it in your own `main.go` and ship a single custom binary (requires Go 1.25+):

```go
package main

import (
    "log"

    "github.com/pocketbase/pocketbase"
    "github.com/pocketbase/pocketbase/core"
)

func main() {
    app := pocketbase.New()

    app.OnServe().BindFunc(func(se *core.ServeEvent) error {
        se.Router.GET("/hello", func(re *core.RequestEvent) error {
            return re.String(200, "Hello world!")
        })
        return se.Next()
    })

    if err := app.Start(); err != nil {
        log.Fatal(err)
    }
}
```

```sh
go mod init myapp && go mod tidy
CGO_ENABLED=0 go build   # statically linked, no C toolchain needed
./myapp serve
```

Client apps talk to it through the official SDKs: [pocketbase/js-sdk](https://github.com/pocketbase/js-sdk) (browser, Node, React Native) and [pocketbase/dart-sdk](https://github.com/pocketbase/dart-sdk).

## Architecture / How It Works

PocketBase is a Go application composed around a `core.App` object and an event-hook system. Requests, record CRUD, auth, and lifecycle all emit hooks (`OnServe`, `OnRecordCreate`, and so on) that you bind functions to — this is the primary extension seam in both Go and JS.

- **Database** — SQLite accessed through a pure-Go, CGO-free driver, which is why `CGO_ENABLED=0` builds work and cross-compilation to the many listed targets (linux/arm, riscv64, s390x, windows/arm64, etc.) is trivial. Data lives in a `pb_data/` directory alongside the binary, not inside it — "one file" refers to the executable, not the persisted state. The schema is defined through "collections" (tables) managed in the admin UI or via migrations.
- **Admin UI** — a compiled single-page app embedded into the Go binary via `embed`, served at `/_/`. Editing collections in the UI mutates the underlying SQLite schema.
- **Realtime** — delivered over **Server-Sent Events (SSE)**, not WebSockets. Clients subscribe to a collection or record; the server pushes create/update/delete events. Because it is SSE over one long-lived HTTP connection per client, the concurrency profile is bounded by open connections and the single-node process.
- **Auth** — built-in email/password, OAuth2 providers, and OTP, with records stored in auth collections. JWT-based; no external identity provider required.
- **JavaScript extensibility** — the prebuilt binaries embed a JS VM (an ECMAScript engine implemented in Go) so you can add hooks, routes, and migrations in `pb_hooks/*.js` without recompiling. This is the no-Go path to customization; it trades raw performance and type safety for zero-build iteration.

The Go-library mode and the standalone-binary mode are the same codebase — the prebuilt executable is just `examples/base/main.go` with the JS VM enabled.

## Production Notes

- **Single node is the operating model.** There is no supported multi-writer or clustered deployment. Plan for one instance plus a good backup and failover story, not a horizontally-scaled fleet. Read replicas / litestream-style streaming backup are community patterns, not first-party features.
- **Back up `pb_data/`, and do it while respecting SQLite.** PocketBase has a built-in backup feature; naive `cp` of a live SQLite file can capture a torn write. Use the built-in backups or a SQLite-aware tool.
- **SQLite write contention is the scaling limit.** Under concurrent writes you will see the single-writer serialize work. WAL mode helps read concurrency but not write throughput. If your workload is write-heavy and fan-out, PocketBase is the wrong shape.
- **Pre-1.0 upgrades are migrations.** Read the release notes for every bump. The v0.23 refactor changed hook names and core APIs; JS hooks and Go extension code written against older versions needed rewrites[^3]. Pin your version and test upgrades against a copy of production data.
- **File storage** defaults to the local filesystem under `pb_data/`; S3-compatible object storage is configurable and is the practical choice once you have more than one deployment target or ephemeral disks.
- **Contribution model is deliberately narrow.** The maintainer asks that feature PRs be discussed first, closes well-executed PRs that fall outside the roadmap, and — due to LLM spam — has temporarily restricted PRs to existing collaborators[^4]. This keeps the project coherent but means external velocity depends heavily on one person.

## When to Use / When Not

**Use when:**
- You want a complete backend (auth + DB + realtime + files + admin) running in minutes with no infrastructure.
- You're building a prototype, indie SaaS, internal tool, or mobile app backend where a single node is plenty.
- You value a single portable binary and CGO-free cross-compilation.
- You want to extend a backend in Go or drop-in JS without standing up microservices.

**Avoid when:**
- You need horizontal scaling, multi-region, or high concurrent write throughput — SQLite single-node is a hard limit.
- You require strict backward-compatibility guarantees; PocketBase is pre-1.0 by its own statement.
- Your team needs Postgres-specific features, a large managed cloud, or many-hands contribution throughput.
- You need a vendor SLA or a broad plugin marketplace rather than a focused single-maintainer project.

## Alternatives

- supabase/supabase — use instead when you need Postgres, row-level security, horizontal scale, and a managed cloud with a team behind it.
- appwrite/appwrite — use instead when you want a self-hosted, Docker-based BaaS with broader language SDK and service coverage.
- directus/directus — use instead when you need an admin/data platform layered over an existing SQL database (including Postgres/MySQL).
- nhost/nhost — use instead when you want a GraphQL-first BaaS built on Postgres + Hasura.
- surrealdb/surrealdb — use instead when you want a multi-model database with built-in auth and realtime as the primitive, not a batteries-included app server.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2022-07 | First public release (Show HN). Single-binary SQLite backend with admin UI[^1]. |
| 0.x | 2022–2024 | Iterative pre-1.0 development; auth providers, OAuth2, JS VM plugin, file storage. |
| 0.23.0 | 2024-11 | Large backward-incompatible refactor of the core Go API and event hooks (`OnServe`/`BindFunc`)[^3]. |
| 0.x | 2025–2026 | Continued pre-1.0 releases; Go 1.25+ toolchain, ongoing API refinement. Still no v1.0 / no compat guarantee[^2]. |

## References

[^1]: PocketBase — official site and documentation. https://pocketbase.io
[^2]: PocketBase README, active-development / no-backward-compatibility warning. https://github.com/pocketbase/pocketbase#readme
[^3]: PocketBase releases and changelog (v0.23 API refactor). https://github.com/pocketbase/pocketbase/releases
[^4]: PocketBase README, contributing section (roadmap-gated PRs; LLM-spam PR restriction). https://github.com/pocketbase/pocketbase#contributing

## Tags

go, sqlite, backend-as-a-service, realtime, authentication, self-hosted, single-binary, embedded-database, admin-ui, rest-api
