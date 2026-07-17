# ClickHouse/clickhouse-connect

> The official ClickHouse Python driver, built on the HTTP interface, with first-class Pandas/Arrow/Polars and Apache Superset integration.

[GitHub repo](https://github.com/ClickHouse/clickhouse-connect) ·
[Documentation](https://clickhouse.com/docs/integrations/python) ·
[License: Apache-2.0](https://github.com/ClickHouse/clickhouse-connect/blob/main/LICENSE)

## Overview

ClickHouse Connect is the officially maintained Python driver for ClickHouse, developed inside ClickHouse Inc. rather than the community[^1]. Its scope is deliberately data-engineering-shaped: it exists to move large result sets and bulk inserts between ClickHouse and the Python analytics stack — Pandas DataFrames (NumPy- and Arrow-backed), NumPy arrays, PyArrow tables, and Polars DataFrames — plus a Superset connector and a limited SQLAlchemy dialect. It is not a general-purpose ORM driver.

The defining design decision is that ClickHouse Connect talks to the server over the **HTTP interface** (ports 8123/8443), not the native TCP protocol (port 9000) used by the older community driver `clickhouse-driver`[^2]. HTTP is chosen for maximum compatibility: it traverses load balancers, reverse proxies, and ClickHouse Cloud endpoints cleanly, and it is the transport ClickHouse Cloud exposes by default. The historical objection to HTTP — that it is slower than the native protocol — is mitigated by transferring result data in ClickHouse's compact `Native`/`RowBinary` column format over HTTP and parsing it in a C extension, rather than parsing HTTP-shaped rows. The tradeoff is real but narrow: for latency-sensitive, tiny-query workloads the native protocol still has an edge; for analytical bulk transfer the difference is usually dominated by serialization and compression, not transport framing.

The library requires Python 3.10 or higher, and Pandas 2.0 or later for DataFrame support[^1]. The 1.0 release introduced breaking API changes from the 0.x line, documented in `MIGRATION.md`[^3].

## Getting Started

```bash
pip install clickhouse-connect
# optional extras:
pip install clickhouse-connect[async]     # aiohttp-based async client
pip install clickhouse-connect[alembic]   # Alembic migration support
```

```python
import clickhouse_connect

client = clickhouse_connect.get_client(
    host="localhost", port=8123, username="default", password=""
)

# Query straight into a Pandas DataFrame
df = client.query_df("SELECT number, number * 2 AS doubled FROM numbers(1000)")

# Column-oriented bulk insert (fast path — pass columns, not row dicts)
client.insert(
    "events",
    data=[[1, 2, 3], ["a", "b", "c"]],
    column_names=["id", "name"],
    column_oriented=True,
)
```

```python
# Async client (aiohttp) — same query surface, awaitable
import asyncio, clickhouse_connect

async def main():
    client = await clickhouse_connect.get_async_client(host="localhost")
    result = await client.query("SELECT count() FROM system.tables")
    print(result.result_rows)

asyncio.run(main())
```

## Architecture / How It Works

The client is an HTTP client wrapping ClickHouse's query/insert endpoints, but the performance-critical parts are not in Python. Data serialization and deserialization for ClickHouse's binary column formats are implemented in a **Cython/C extension**, with a pure-Python fallback used when the compiled extension is unavailable[^1]. Query results are requested in a column-native binary format and materialized directly into NumPy/Arrow buffers where possible, which is what makes `query_df` / `query_arrow` cheaper than reconstructing rows.

Key surfaces:

- **Query methods** — `query` (native `QueryResult` with typed rows/columns), `query_df` (Pandas), `query_arrow` (PyArrow), `query_np` (NumPy), plus streaming variants (`query_row_block_stream`, `query_df_stream`) that yield blocks so large results do not have to fit in memory at once.
- **Insert** — column-oriented insert is the fast path; row-oriented insert is supported but pays a transposition cost. Inserts stream data to the server rather than building one giant request body.
- **Type system** — a Python-side type registry maps ClickHouse types (including `LowCardinality`, `Enum`, `Nullable`, `Array`, `Tuple`, `Map`, `Decimal`, and the various date/time widths) to Python/NumPy/Arrow equivalents. Type fidelity across these three targets is not identical, which surfaces as the most common class of user surprises.
- **Async client** — a parallel implementation over `aiohttp` sharing the same query surface. It is genuinely async I/O, not a thread-pool wrapper.
- **SQLAlchemy dialect** — a lightweight dialect (`clickhousedb://` DSN) scoped to SQLAlchemy Core and Superset, supporting both SQLAlchemy 1.4 and 2.x. 1.4 is kept alive specifically because Apache Superset still pins `sqlalchemy>=1.4,<2`[^1].

The coupling story is with ClickHouse Cloud and Superset, not with a hosting platform: the HTTP-first choice and the retained SQLAlchemy 1.4 support both exist to serve those two consumers.

## Production Notes

**Column-oriented inserts matter.** Passing row dicts or lists of rows forces a transpose into columns before serialization. For high-throughput ingestion, pass `column_oriented=True` with columnar data (or insert a DataFrame/Arrow table directly). This is the single biggest self-inflicted performance issue.

**Server settings travel with the query.** ClickHouse behaviors like `max_execution_time`, `async_insert`, and `insert_deduplicate` are passed via the `settings` argument on query/insert calls, not via connection state. Forgetting this is a common cause of "works in clickhouse-client, fails here" reports.

**Type round-tripping is not free.** The NumPy, Arrow, and Polars paths each map ClickHouse types slightly differently — nullability, timezone-aware `DateTime`, `UInt64` above 2^63, and `Decimal` precision are the usual friction points. Verify types on both ends when a pipeline crosses formats; do not assume `query_df` and `query_arrow` yield identical schemas.

**SQLAlchemy is not an ORM path.** The dialect supports Core queries, `JOIN`s (with ClickHouse strictness/`USING`/`GLOBAL`), `FINAL`, `SAMPLE`, `VALUES`, lightweight `DELETE`, and Alembic migrations. It does not implement `UPDATE` compilation, foreign-key/relationship reflection, autoincrement/`RETURNING`, or cascades[^1]. Insert-heavy, read-focused ORM usage works; a relational ORM app will hit walls.

**Superset version coupling.** The dynamically loaded Superset engine spec was removed in clickhouse-connect v0.6.0 once Apache Superset 2.1.0 absorbed it upstream. Connecting to Superset older than 2.1.0 requires pinning clickhouse-connect v0.5.25[^1]. This is an easy-to-miss compatibility cliff for teams on older Superset.

**Upgrading 0.x → 1.0 is a breaking change.** The 1.0 line changed public APIs; read `MIGRATION.md` before bumping[^3]. Also note the hard floors: Python 3.10 and Pandas 2.0 — both can block upgrades on older platforms.

## When to Use / When Not

**Use when:**
- You are moving data between ClickHouse and Pandas/Arrow/Polars/NumPy in Python.
- You target ClickHouse Cloud or run behind proxies/load balancers where HTTP is the practical transport.
- You need a Superset data source or SQLAlchemy Core access to ClickHouse.
- You want an officially maintained driver tracking current ClickHouse types and features.

**Avoid when:**
- You need a full SQLAlchemy ORM with updates, relationships, and cascades — the dialect is Core/Superset-scoped.
- You are on Python < 3.10 or cannot move to Pandas 2.0.
- Your workload is many tiny low-latency queries where the native TCP protocol's framing advantage matters.

## Alternatives

- mymarilyn/clickhouse-driver — community driver over the native TCP protocol (port 9000); use when you prefer the native wire protocol or need behaviors the HTTP path does not expose.
- xzkostyan/clickhouse-sqlalchemy — fuller SQLAlchemy dialect built on clickhouse-driver; use when you need richer dialect/ORM support than the Superset-focused dialect here.
- ClickHouse/clickhouse-go — official Go driver; use for Go services rather than Python.
- ClickHouse/clickhouse-js — official JS/TS driver; use for Node/browser clients.
- apache/superset — if your only goal is dashboards, connect Superset directly (which now bundles the ClickHouse engine spec) instead of scripting the driver.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2022-02 | First public release; HTTP-based driver with a Cython/C-extension core[^1]. |
| 0.5.25 | 2023 (approx) | Last release carrying the dynamic Superset engine spec (pin for Superset < 2.1.0)[^1]. |
| 0.6.0 | 2023 (approx) | Superset engine spec removed after Apache Superset 2.1.0 absorbed it upstream[^1]. |
| 1.0 | 2025 (approx) | Breaking API changes from 0.x; Python 3.10+ and Pandas 2.0+ required[^3]. |

## References

[^1]: ClickHouse Connect README and ClickHouse integration docs. https://clickhouse.com/docs/integrations/python
[^2]: ClickHouse HTTP interface documentation. https://clickhouse.com/docs/interfaces/http
[^3]: clickhouse-connect MIGRATION.md (0.x → 1.0 breaking changes). https://github.com/ClickHouse/clickhouse-connect/blob/main/MIGRATION.md

## Tags

python, clickhouse, database-driver, olap, pandas, apache-arrow, polars, superset, sqlalchemy, data-engineering, analytics
