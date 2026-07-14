# postgres/postgres

> The PostgreSQL relational database — an object-relational SQL engine with a 30-year track record for correctness and extensibility.

[GitHub repo](https://github.com/postgres/postgres) ·
[Official website](https://www.postgresql.org/) ·
[License: PostgreSQL](https://www.postgresql.org/about/licence/)

## Overview

PostgreSQL is an object-relational database management system descended from the POSTGRES project led by Michael Stonebraker at UC Berkeley in the mid-1980s; the SQL layer and the "PostgreSQL" name arrived in 1996[^1]. It implements a large subset of the SQL standard plus a deep set of extensions: user-defined types and functions, table inheritance, rich indexing (B-tree, GiST, GIN, BRIN, SP-GiST, Hash), MVCC transactions, and procedural languages (PL/pgSQL, PL/Python, PL/Perl). It is the default choice when a project wants strict correctness, complex queries, and room to grow schema complexity.

This GitHub repository is explicitly a **mirror**. Development happens on the `pgsql-hackers` mailing list with patches reviewed through the CommitFest process; the project does not accept pull requests here[^2]. The star and fork counts on GitHub therefore reflect popularity, not contribution flow — the real bug tracker, discussion, and review live off-platform.

The defining tension is between PostgreSQL's correctness-first, standards-leaning conservatism and the operational realities of its MVCC storage design. You get a database that rarely surprises you at the SQL semantics level, but whose row-versioning model imposes vacuum, bloat, and connection-scaling concerns that operators must actively manage.

## Getting Started

```bash
# Debian/Ubuntu
sudo apt install postgresql

# macOS (Homebrew)
brew install postgresql@17 && brew services start postgresql@17

# Docker
docker run -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:17
```

```sql
-- Connect: psql -h localhost -U postgres
CREATE TABLE users (
    id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email  text NOT NULL UNIQUE,
    data   jsonb NOT NULL DEFAULT '{}',
    added  timestamptz NOT NULL DEFAULT now()
);

INSERT INTO users (email, data) VALUES ('tom@example.com', '{"plan":"pro"}');

-- JSONB is indexable and queryable, not just a blob
CREATE INDEX idx_users_data ON users USING gin (data);
SELECT email FROM users WHERE data @> '{"plan":"pro"}';
```

## Architecture / How It Works

PostgreSQL uses a **process-per-connection** model. The `postmaster` supervisor forks one backend process per client connection; there is no built-in connection multiplexing. This makes each connection heavyweight (each backend has its own memory and a corresponding entry in shared structures), which is why external poolers are near-mandatory at scale.

**MVCC (Multi-Version Concurrency Control)** is the core of the storage engine. An `UPDATE` does not overwrite a row in place — it writes a new row version (tuple) and marks the old one dead. Readers see a consistent snapshot without blocking writers. The cost is that dead tuples accumulate and must be reclaimed by **VACUUM**, and every table needs periodic vacuuming both to reclaim space and to prevent transaction-ID wraparound.

Writes go through the **WAL (Write-Ahead Log)** first for durability and crash recovery; the WAL is also the substrate for streaming replication and logical replication[^3]. A checkpoint periodically flushes dirty shared-buffer pages to the heap.

The **planner/optimizer** is cost-based, relying on statistics gathered by `ANALYZE`. It supports a wide index arsenal and extension-defined index access methods. **Extensibility** is a first-class design property: extensions like PostGIS (geospatial), pgvector (vector similarity), TimescaleDB, and `pg_stat_statements` hook into custom types, index methods, and background workers without forking the core.

## Production Notes

**Connection scaling is the classic footgun.** Because each connection is a process, a few hundred idle connections consume real memory and lock/snapshot overhead. Put a pooler (PgBouncer in transaction mode, or the newer pgcat) in front of anything with a large or bursty client count. Serverless/edge apps that open a connection per request will exhaust `max_connections` quickly without one.

**VACUUM and bloat.** Autovacuum is on by default but its defaults are tuned conservatively for small machines. High-churn tables (queues, counters, frequently-updated hot rows) bloat if autovacuum falls behind; symptoms are growing table/index size and slowing scans. Tune `autovacuum_vacuum_scale_factor` per-table and monitor dead-tuple counts. **Transaction-ID wraparound** is the more dangerous version: if vacuum never freezes old tuples, the database will force a shutdown to protect data — this has caused real outages at large shops.

**Major-version upgrades are not in-place by default.** Moving from e.g. 16 to 17 requires either `pg_dump`/restore or `pg_upgrade` (which relinks data files, faster but with its own checklist). Logical replication enables lower-downtime upgrades but adds setup complexity. Minor releases (17.1 → 17.2) are binary-compatible and low-risk; apply them promptly for security fixes.

**Replication caveats.** Physical streaming replication is byte-level and requires identical major versions; logical replication is per-table and version-flexible but historically did not replicate schema (DDL) changes, sequences state, or large objects automatically — verify what your version covers before relying on it.

**Defaults worth changing.** `shared_buffers` defaults are low; `work_mem` is per-operation (a single query with many sorts/hashes can multiply it); `default_statistics_target` may need raising for skewed data. Long-running transactions hold back vacuum globally — watch for idle-in-transaction sessions.

## When to Use / When Not

**Use when:**
- You need transactional correctness, foreign keys, and complex multi-table queries.
- Your schema will grow in complexity (JSONB, arrays, custom types, full-text search, geospatial via PostGIS).
- You want one engine that covers OLTP well and moderate analytics acceptably.
- You value a permissive license and no single-vendor control.

**Avoid when:**
- You need a horizontally-sharded write-scalable cluster out of the box (Postgres scales reads via replicas; write-sharding needs Citus or app-level partitioning).
- Your workload is pure high-volume analytics/columnar scans — a column store (ClickHouse, DuckDB) will be far faster.
- You need a schemaless document store as the primary model and reject relational constraints entirely.
- You cannot manage vacuum/connection operations and have no managed provider.

## Alternatives

- mysql/mysql-server — the other default open-source RDBMS; simpler operationally in some respects, less strict SQL and fewer advanced types.
- cockroachdb/cockroach — Postgres wire-compatible with built-in horizontal scaling and geo-distribution, at higher latency and operational cost.
- sqlite/sqlite — use instead when you need an embedded, single-file database with no server process.
- ClickHouse/ClickHouse — use instead for columnar OLAP and high-ingest analytics, not transactional workloads.
- duckdb/duckdb — use instead for embedded analytical queries over local files (the "SQLite for analytics").

## History

| Version | Date | Notes |
|---------|------|-------|
| POSTGRES | 1986 | Berkeley research project (Stonebraker)[^1]. |
| Postgres95 / 6.0 | 1996 | SQL added; renamed PostgreSQL[^1]. |
| 8.0 | 2005-01 | Native Windows support, savepoints, PITR. |
| 9.0 | 2010-09 | Streaming replication, hot standby. |
| 9.4 | 2014-12 | JSONB, logical decoding foundation. |
| 10 | 2017-10 | Native logical replication, declarative partitioning, new versioning scheme. |
| 12 | 2019-10 | Generated columns, pluggable table storage (AM) API. |
| 14 | 2021-09 | Pipeline mode, multirange types, JSONB subscripting. |
| 16 | 2023-09 | Parallelism and logical replication improvements. |
| 17 | 2024-09 | Incremental backup, improved vacuum memory management[^4]. |

## References

[^1]: PostgreSQL Global Development Group, "A Brief History of PostgreSQL." https://www.postgresql.org/docs/current/history.html
[^2]: PostgreSQL Wiki, "Submitting a Patch" — describes the mailing-list / CommitFest workflow the GitHub mirror does not replace. https://wiki.postgresql.org/wiki/Submitting_a_Patch
[^3]: PostgreSQL documentation, "Write-Ahead Logging (WAL)." https://www.postgresql.org/docs/current/wal-intro.html
[^4]: PostgreSQL Global Development Group, "PostgreSQL 17 Released" — 2024-09-26. https://www.postgresql.org/about/news/postgresql-17-released-2936/

## Tags

database, sql, relational-database, postgresql, rdbms, mvcc, c, acid, transactions, storage-engine
