# geldata/gel

> A graph-relational database that runs EdgeQL on top of PostgreSQL — formerly EdgeDB.

[GitHub repo](https://github.com/geldata/gel) ·
[Official website](https://geldata.com) ·
[License: Apache-2.0](https://github.com/geldata/gel/blob/master/LICENSE)

## Overview

Gel is a database server built by MagicStack (Yury Selivanov and Elvis Pranskevichus) that stores data in PostgreSQL but exposes a different data model and a different query language[^1]. Instead of tables, columns, and foreign keys, you declare **object types** with **properties** and **links**; instead of SQL you write **EdgeQL**, a language whose queries return nested, structured objects rather than flat rows. The project was called EdgeDB from its 2017 open-sourcing through 2024, and was renamed **Gel** in early 2025 along with a broader push into built-in auth, AI/vector, and first-class SQL access[^2].

The defining bet is that the relational model's tabular surface is the wrong abstraction for application developers, and that a purpose-built layer over Postgres can give you graph-style traversal, a real schema/migration system, and generated typed clients without you assembling an ORM, a migration tool, and a query builder yourself. The defining tension is the flip side of that bet: Gel is a **new query language and a new server process**, not a library or a Postgres extension. You adopt EdgeQL, the `gel` toolchain, and an operational model where a Gel server owns a Postgres backend — real lock-in at the language and tooling layer even though your bytes live in Postgres.

Gel is Apache-2.0 and self-hostable; the company monetizes through Gel Cloud, a managed hosting service[^3]. The server is written primarily in Python (with Cython and Rust in the hot paths) and compiles EdgeQL down to SQL that Postgres executes.

## Getting Started

Install the CLI, initialize a project, and open an interactive shell:

```bash
# macOS / Linux
curl --proto '=https' --tlsv1.2 -sSf https://geldata.com/sh | sh

gel project init      # provisions a local instance, links it to the project
gel                   # interactive EdgeQL shell
```

Define a schema in an `.gel` (ESDL) file, then declaratively migrate:

```esdl
# dbschema/default.gel
type Person {
  required name: str;
}

type Movie {
  required title: str;
  multi actors: Person;
}
```

```bash
gel migration create   # diffs schema against DB, writes a migration
gel migrate            # applies it
```

Query with EdgeQL — note the nested shape, no JOINs:

```edgeql
select Movie {
  title,
  actors: { name }
}
filter .title = "The Matrix";
```

Older material and the CLI's own history still use the `edgedb` command and `.esdl` extension; the `gel` names are the current ones and both may appear in the wild during the rebrand transition[^2].

## Architecture / How It Works

Gel is a server that sits in front of PostgreSQL. It does not reimplement storage, MVCC, or the planner — Postgres does the actual work[^1]. What Gel adds is a compilation and protocol layer:

- **Schema layer.** Your ESDL schema is stored and versioned inside the database. Object types map to Postgres tables, links map to columns or link tables, and the migration system diffs declarative schema states to produce ordered, hashed migration scripts rather than hand-written DDL.
- **EdgeQL compiler.** Each EdgeQL query is parsed and compiled to one or more SQL statements. The nested object shapes you request are assembled in SQL (largely via JSON aggregation and lateral joins) so a single EdgeQL query with deep fetches becomes a single round trip.
- **Binary protocol.** Clients speak Gel's own binary protocol, not the Postgres wire protocol. Official client libraries (TypeScript/JS, Python, Go, Rust, .NET, Elixir, and others) build on it, and the TS/JS ecosystem gets a generated, fully typed query builder.
- **SQL passthrough.** Since EdgeDB 5 / Gel 6 the server also answers a subset of the **PostgreSQL wire protocol**, so BI tools and raw SQL `select` queries can read the underlying tables directly[^4]. This is a deliberate escape hatch against total lock-in.

Later versions added **branches** (5.0) — lightweight, git-like database branches for schema development that replaced the older "databases" concept[^5] — and extension modules shipped in-tree: `ext::auth` for email/OAuth authentication and `ext::ai` for embeddings and retrieval-augmented queries. These are what the current tagline means by "Auth & AI solutions"; they run inside the Gel server rather than as separate services.

Because everything compiles to Postgres, Gel inherits Postgres's transactional guarantees and much of its performance profile — but also inherits its ceilings, and adds a compilation step and a second process to operate.

## Production Notes

- **It is a server, not an embedded library.** You run and monitor a Gel process plus its Postgres backend. For local dev the CLI can manage a bundled Postgres; in production you either self-manage both or use Gel Cloud. This is heavier operationally than "just point an ORM at RDS."
- **Migrations are declarative and can require decisions.** `gel migration create` diffs schema states and will interactively ask how to resolve ambiguous changes (e.g. is a renamed property a rename or a drop+add). Automating migrations in CI means pre-answering these or scripting them; a surprising diff can block a deploy.
- **The query language is a real adoption cost.** EdgeQL is expressive but non-portable. Team ramp-up, hiring, LLM assistance, and third-party tooling are all thinner than for SQL. The SQL passthrough helps for reads and reporting but is not a full substitute for writes.
- **Client/server version coupling.** Client libraries, the CLI, and the server advance together; mixing a much newer CLI with an older server (or vice versa) can produce protocol or migration-format mismatches. Pin versions across your fleet.
- **Rebrand churn.** Package names, the CLI binary, environment variables, and docs moved from `edgedb` to `gel` across 2024–2025. Expect stale search results, split documentation, and mixed naming in issues and blog posts for a while[^2].
- **Postgres is the real substrate.** Connection limits, autovacuum, index bloat, and query planning are still Postgres concerns. Deep EdgeQL shapes compile to large SQL, so `analyze`-style inspection of the generated SQL matters when a query is slow.

## When to Use / When Not

**Use when:**
- You want graph-style, deeply-nested reads and rich schema modeling without hand-building an ORM + migrations + query builder.
- You value a strongly-typed, generated client (especially TypeScript) as a first-class feature.
- You're starting a new application and are willing to adopt EdgeQL and the `gel` toolchain end to end.
- You want built-in auth and AI/vector features living next to your data.

**Avoid when:**
- You need broad ecosystem compatibility — existing SQL, ORMs, BI tools, DBAs, and hiring pools assume plain Postgres.
- You're integrating into an established Postgres estate and don't want a second server process or a new query language.
- Your team can't absorb the learning curve of a proprietary query language and migration model.
- You need the maturity and battle-testing of a decades-old engine for a high-stakes system of record.

## Alternatives

- supabase/supabase — if you want the "batteries-included Postgres platform" (auth, storage, realtime, hosting) but with plain Postgres and SQL rather than a new language.
- prisma/prisma — if you want a typed schema and generated client over Postgres without adopting a new server or query language.
- hasura/graphql-engine — if the appeal is graph-shaped fetching over Postgres and you'd rather expose GraphQL than learn EdgeQL.
- surrealdb/surrealdb — if you specifically want a from-scratch multi-model graph-relational database and are comparing new-database bets.
- postgres/postgres — if the honest answer is that plain Postgres with a thin client already covers your needs.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-sourced | 2017-06 | EdgeDB repository first published on GitHub[^1]. |
| Alpha/Beta | 2019–2021 | Public alphas and betas; EdgeQL and the schema/migration system stabilize. |
| 1.0 | 2022-02 | First stable EdgeDB release[^6]. |
| 2.0 | 2022-07 | GUI data explorer, analytics, expanded stdlib. |
| 3.0–4.0 | 2023 | Query performance, `ext::auth` and AI groundwork, tooling. |
| 5.0 | 2024 | Branches replace multi-database model; SQL read protocol support[^5]. |
| Gel 6.0 | 2025 | Rebrand from EdgeDB to Gel; `gel` CLI, expanded auth/AI, SQL access[^2]. |

## References

[^1]: geldata/gel README and repository history. https://github.com/geldata/gel
[^2]: Gel blog, "EdgeDB is now Gel" (rebrand announcement, 2025). https://www.geldata.com/blog
[^3]: Gel Cloud — managed hosting service. https://www.geldata.com/cloud
[^4]: Gel documentation, SQL support / PostgreSQL-protocol access. https://docs.geldata.com/reference/sql
[^5]: EdgeDB 5 release notes — branches. https://docs.geldata.com/resources/changelog
[^6]: EdgeDB 1.0 announcement (2022-02). https://www.geldata.com/blog

## Tags

database, postgresql, graph-relational, edgeql, edgedb, orm, query-language, python, schema-migrations, backend, relational-database
