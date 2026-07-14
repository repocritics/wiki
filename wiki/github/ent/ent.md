# ent/ent

> A schema-as-code entity framework for Go that generates a fully type-safe client from Go-defined schemas.

[GitHub repo](https://github.com/ent/ent) ·
[Official website](https://entgo.io) ·
[License: Apache-2.0](https://github.com/ent/ent/blob/master/LICENSE)

## Overview

ent is an ORM/entity framework for Go, open-sourced by Facebook's Connectivity team in 2019 and inspired by "Ent", an entity framework used internally at Meta[^1]. Its defining idea is *schema as code*: you describe each entity as a Go struct implementing `ent.Schema`, run code generation, and get a package of statically typed builders — no struct tags, no reflection at query time, no string-keyed query DSL. Every field, edge, and predicate becomes a Go symbol the compiler checks.

The data model is a graph. Entities are nodes; relationships are first-class *edges* with inverse sides, and the generated query API is built around traversing them (`WithPets()`, `QueryPets()`). This graph framing — rather than SQL's table/JOIN framing — is what most distinguishes ent from reflection ORMs like GORM, and it is the thing new users most often have to unlearn SQL habits for.

The central tradeoff is code generation. In exchange for compile-time safety and autocomplete over your entire schema, you accept a generated `ent/` package that is checked into the repo, must be regenerated after every schema change, and produces large diffs. Runtime-dynamic queries (where columns or predicates aren't known at compile time) are awkward by design. ent is developed and maintained today by the [Atlas](https://atlasgo.io) team at Ariga, and joined the Linux Foundation in 2021[^2]. Despite heavy production use it remains pre-1.0; the v1 roadmap is tracked in issue #46[^3].

## Getting Started

```bash
go install entgo.io/ent/cmd/ent@latest
ent new User Pet          # scaffold ent/schema/*.go
```

```go
// ent/schema/user.go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/edge"
    "entgo.io/ent/schema/field"
)

type User struct{ ent.Schema }

func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
        field.Int("age").Positive(),
    }
}

func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("pets", Pet.Type),   // one-to-many, Pet declares the inverse
    }
}
```

```bash
go generate ./ent          // regenerate the typed client after any schema change
```

```go
client, _ := ent.Open("sqlite3", "file:ent?mode=memory&_fk=1")
client.Schema.Create(ctx)  // auto-migration (additive only)

client.User.Create().SetName("a8m").SetAge(30).SaveX(ctx)

users := client.User.Query().
    Where(user.AgeGT(18)).
    WithPets().              // eager-load the edge
    AllX(ctx)
```

## Architecture / How It Works

The pipeline has two halves: your hand-written schemas, and the generated runtime client.

1. **Schema definitions** (`ent/schema/*.go`) — plain Go implementing the `ent.Schema` interface, exposing `Fields()`, `Edges()`, `Indexes()`, `Hooks()`, `Policy()`, and mixins. This is the only code you edit.
2. **entc (the code generator)** — invoked via `go generate`. It loads the schema graph, validates it, and renders the client package from Go `text/template`s. Because codegen is template-driven, extensions can inject fields, templates, and annotations — this is how features like the GraphQL (`entgql`), gRPC (`entproto`), and REST integrations plug in.
3. **Generated client** (`ent/`) — one builder set per entity: `Create`, `Update`, `Query`, `Delete`, plus typed predicates (`user.AgeGT`), typed edge loaders, and pagination helpers. The runtime speaks to a SQL dialect layer supporting MySQL, MariaDB, TiDB, PostgreSQL, CockroachDB, and SQLite, plus a Gremlin driver for graph databases.

Three composable middleware layers run on the client, not in the generated builders:

- **Hooks** — mutation middleware (`func(Mutator) Mutator`), wired per-entity or globally; the standard place for audit fields, validation, and side effects.
- **Privacy** — a policy layer of query/mutation rules evaluated in order, returning allow/deny/skip. Enforces row-level access inside ent rather than in application code.
- **Interceptors** — query middleware (added later than hooks), for read-side concerns like soft-delete filtering or multi-tenancy.

Eager loading via `With<Edge>()` issues a **separate query per edge**, not a JOIN — ent batches the child rows and stitches them in memory. This avoids row-multiplication bugs but means an eager query is N+1 queries by construction (bounded, not per-row). Raw escape hatches exist: `Modify()` for query mutation, the `sql/sqljson` predicates, and direct dialect access, but they trade away the type safety that is ent's whole point.

## Production Notes

**Migrations are the biggest operational decision.** `client.Schema.Create()` auto-migration is **additive only** — it creates tables, columns, and indexes but never drops or alters destructively, so it silently ignores renames and removals. For anything real, use **Atlas versioned migrations**[^4]: ent diffs the schema, Atlas writes migration files you review and commit, and you apply them deterministically. Teams that ship auto-migrate to production eventually hit a schema change it can't express and have to retrofit versioned migrations under pressure.

**Generated code is a repo citizen.** The `ent/` package is committed, so schema changes produce large mechanical diffs, and reviewers must learn to skim them. CI should run `go generate ./...` and fail if the tree is dirty, to catch schemas that were edited without regenerating.

**Version coupling.** The `entgo.io/ent` runtime version and the `ent` codegen binary should match; regenerating with a mismatched `cmd/ent` can produce a client that won't compile against the pinned runtime. Pin codegen with a tools file (`//go:build tools`) rather than a globally installed binary.

**Dynamic queries fight the grain.** Anything where the filtered field or sort column is decided at runtime (generic admin filters, user-defined search) is verbose in ent — you assemble predicates via `sql.P()` or fall back to raw SQL. If most of your queries are dynamic, ent's typing buys you little.

**Compile and codegen time** grow with schema size; a few hundred entities produce a large generated package that measurably slows `go build`. Splitting unrelated schemas across modules helps.

## When to Use / When Not

**Use when:**
- You have a large, relationship-heavy data model and want compile-time guarantees over every query.
- You prefer generated, explicit code to reflection-and-struct-tags magic.
- You want one framework to also emit a GraphQL or gRPC layer from the same schema.
- Your queries are mostly known at compile time.

**Avoid when:**
- The app is simple CRUD where an ORM's ceremony outweighs its safety — plain `database/sql`, sqlx, or sqlc are lighter.
- You need runtime-dynamic schemas or queries as the common case.
- Your team rejects a code-generation step in the build/review loop.
- You want a stable 1.0 API contract — ent is still pre-1.0 and evolves.

## Alternatives

- go-gorm/gorm — use instead when you want the most popular, reflection-based Go ORM with no codegen step and are willing to trade compile-time safety for flexibility.
- kyleconroy/sqlc — use instead when you're SQL-first: write queries in SQL, generate typed Go, no schema-as-code graph.
- volatiletech/sqlboiler — use instead when your database schema already exists and you want an ORM generated by introspecting the live DB (database-first, not schema-first).
- uptrace/bun — use instead when you want a lightweight SQL-first query builder with struct mapping rather than a full generated client.
- jmoiron/sqlx — use instead when you want thin, explicit extensions over `database/sql` with no ORM abstraction at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-06 | Open-sourced by Facebook Connectivity; created by a8m and alexsn[^1]. |
| — | 2021 | Project joined the Linux Foundation; Atlas (Ariga) team becomes maintainer[^2]. |
| 0.x | 2021–2022 | Hooks, Privacy policies, and Atlas-based versioned migrations added[^4]. |
| 0.x | 2023–2024 | Interceptors (query middleware), extension ecosystem (entgql, entproto) matured. |
| pre-1.0 | ongoing | Still versioned v0.x; v1 roadmap tracked in issue #46[^3]. |

## References

[^1]: ent README, "About the Project" — inspired by Meta's internal Ent, created by a8m and alexsn on the Facebook Connectivity team. https://github.com/ent/ent
[^2]: entgo.io blog, "ent joins the Linux Foundation" (2021). https://entgo.io/blog/2021/06/28/ent-joins-linux-foundation
[^3]: ent issue #46, "ent v1 roadmap". https://github.com/ent/ent/issues/46
[^4]: entgo.io docs, "Versioned Migrations" (Atlas integration). https://entgo.io/docs/versioned-migrations

## Tags

go, orm, entity-framework, code-generation, database, sql, schema-as-code, graph, postgresql, mysql, backend
