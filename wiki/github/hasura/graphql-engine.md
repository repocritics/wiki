# hasura/graphql-engine

> Instant GraphQL (and REST) APIs generated from your database schema, with row-level authorization compiled into SQL.

[GitHub repo](https://github.com/hasura/graphql-engine) ·
[Official website](https://hasura.io) ·
[License: Apache-2.0](https://github.com/hasura/graphql-engine/blob/master/LICENSE)

## Overview

Hasura points at an existing database and generates a GraphQL API over its
tables, views, and relationships without you writing resolvers. Queries,
mutations, and subscriptions are derived from the schema plus a metadata layer
that declares relationships, permissions, and remote joins. The value
proposition is time-to-API: a new Postgres table becomes queryable, filterable,
paginated, and authorized in minutes.

The project is really two engines in one monorepo. **V2** is the mature,
widely-deployed Haskell engine (in `v2/`), the current stable line for most
users. **V3** (in `v3/`) is a newer Rust engine that powers Hasura DDN (Data
Delivery Network) and introduces a connector-based architecture (NDC — Native
Data Connectors) plus a new metadata format (OpenDD/HML)[^1]. These are not a
drop-in upgrade path for each other; adopting V3 is a re-platforming, not a
`docker pull`.

The defining tension is the open-core boundary. The engine itself is Apache-2.0,
but many features operators reach for in production — query response caching,
rate limiting, per-role API limits, read-replica routing, and enforced
allow-lists — are gated behind Hasura Cloud or the Enterprise Edition[^2]. The
OSS engine is genuinely useful, but "Hasura in production" and "the open-source
binary" are not the same product.

## Getting Started

The V2 engine runs as a single container against a Postgres URL:

```bash
docker run -d -p 8080:8080 \
  -e HASURA_GRAPHQL_DATABASE_URL=postgres://user:pass@host:5432/mydb \
  -e HASURA_GRAPHQL_ENABLE_CONSOLE=true \
  -e HASURA_GRAPHQL_ADMIN_SECRET=changeme \
  hasura/graphql-engine:latest
```

Open `http://localhost:8080/console`, track a table, then query it:

```graphql
query {
  users(where: { age: { _gte: 18 } }, order_by: { created_at: desc }, limit: 10) {
    id
    name
    articles(where: { published: { _eq: true } }) {   # relationship traversal
      title
    }
  }
}
```

Permissions are declared per role as boolean expressions over session variables
(e.g. `{ "id": { "_eq": "X-Hasura-User-Id" } }`); the engine folds them into the
generated SQL `WHERE` clause rather than filtering in application code.

## Architecture / How It Works

The core trick is **query compilation, not resolver execution**. An incoming
GraphQL operation — including nested relationships — is compiled into a single
SQL statement (Postgres builds the JSON response via `json_agg`/`row_to_json`),
so a three-level nested query is one round trip to the database, not an N+1
resolver cascade. This is why Hasura's read performance tracks the database's
own rather than a Node resolver layer.

Metadata is the source of truth for everything the schema can't express:
table/view tracking, object and array relationships, role-based permissions,
computed fields, and remote joins. It lives as YAML/JSON managed by the `hasura`
CLI and is applied/migrated separately from your DDL migrations — two migration
streams that must stay in sync.

Beyond plain CRUD, V2 stitches in external logic through several mechanisms:

- **Actions** — wrap a REST/HTTP handler as a GraphQL mutation or query for
  custom business logic.
- **Remote Schemas** — merge another GraphQL server into the same endpoint
  (schema stitching).
- **Event Triggers** — fire a webhook on insert/update/delete, delivered from a
  Postgres-backed event queue with at-least-once semantics.
- **Scheduled / Cron Triggers** — time-based webhook delivery.

V2 supports Postgres and its flavors (Citus, CockroachDB, TimescaleDB, Yugabyte)
plus MS SQL Server, BigQuery, and others. **Subscriptions are live queries
implemented by polling** the underlying query on an interval and multiplexing
identical queries across clients — not push-based change data capture. V3
replaces the built-in DB drivers with out-of-process NDC connectors and a
supergraph model closer to federation.

## Production Notes

**Subscriptions are polling.** Every live query re-runs its SQL on a refresh
interval (default 1s). Hasura multiplexes identical queries, but high-fan-out or
expensive subscription queries put steady load on the primary. Budget for it;
don't assume "realtime" means free.

**Deep queries are a footgun.** Because clients compose arbitrary nesting and
filters, a naive schema lets callers generate expensive multi-join SQL. In
production you want query depth/node limits and, ideally, an **allow-list** of
permitted operations — but allow-list *enforcement* and API rate limits are
Enterprise/Cloud features, not in the OSS binary.

**Permissions live in metadata, not code.** This is powerful (authorization is
centralized and compiled into SQL) and dangerous: a misconfigured role's boolean
expression is a data leak, and it's invisible to your application test suite.
Review permission changes as carefully as schema changes.

**Two migration streams.** Postgres migrations and Hasura metadata are applied
separately via the CLI. Drift between them — a column renamed in SQL but not in
metadata — surfaces as broken relationships or stale permissions. CI should
apply and diff both.

**The engine is memory-hungry.** The Haskell server keeps schema/metadata and
prepared plans resident; large schemas with many roles inflate the introspection
schema (`schema.graphql`) and startup cost. Watch memory on metadata reloads,
which rebuild the schema cache.

**V2 → V3 is a migration project.** V3 (DDN) is a different engine, different
metadata (OpenDD/HML), and a connector model. Treat a move as a re-architecture
with its own timeline, not a version bump. Many V2 deployments are stable and
have no reason to move.

## When to Use / When Not

**Use when:**
- You have a SQL database and want a queryable, authorized API today.
- Row/column-level authorization maps cleanly to session variables and boolean
  rules.
- You want relationships and nested reads without hand-writing resolvers or
  fighting N+1.
- You're consolidating several data sources behind one GraphQL endpoint.

**Avoid when:**
- Your API is mostly custom business logic and workflows — you'll be writing
  Actions and remote schemas for everything, and a purpose-built server is
  simpler.
- You need push-based realtime at scale — polling subscriptions may not fit.
- The open-core boundary is a problem: if you need caching, rate limiting, or
  allow-list enforcement without paying, budget to build them yourself.
- You want the database schema hidden — Hasura's default surface closely mirrors
  your tables.

## Alternatives

- graphile/postgraphile — Postgres-only instant GraphQL, extended via Postgres
  functions and plugins; use when you're single-database and want logic to live
  in the DB.
- supabase/supabase — Postgres backend-as-a-service (auth, storage, PostgREST);
  use when you want a batteries-included REST-first backend rather than
  GraphQL-first generation.
- dgraph-io/dgraph — native GraphQL-first graph database; use when your data is
  genuinely graph-shaped rather than relational.
- apollographql/router — federation/supergraph composition of existing GraphQL
  services; use when you're stitching hand-built subgraphs, not generating from a
  DB.
- directus/directus — headless data platform with REST+GraphQL and an admin UI
  over SQL; use when you also want content workflows and a CMS-style backend.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2018-06 | Repo public; instant GraphQL over Postgres[^3]. |
| 1.0 | 2020-02 | First GA of the Haskell engine. |
| 2.0 | 2021-07 | Multiple databases, MS SQL Server, remote joins, inherited roles[^4]. |
| 2.x | 2021–2024 | BigQuery, Citus/CockroachDB, REST endpoints, streaming subscriptions. |
| 3.0 / DDN | 2024 | Rust engine, NDC connectors, OpenDD metadata, supergraph model[^1]. |

## References

[^1]: Hasura, "Hasura DDN / V3" — README and docs. https://hasura.io/docs/3.0/
[^2]: Hasura, Enterprise Edition / Cloud feature comparison. https://hasura.io/docs/2.0/enterprise/overview/
[^3]: hasura/graphql-engine repository (created 2018-06-18). https://github.com/hasura/graphql-engine
[^4]: Hasura blog, "Hasura 2.0". https://hasura.io/blog/hasura-2-0/

## Tags

graphql, api, postgres, database, backend, haskell, rust, authorization, realtime, open-core, sql, ndc
