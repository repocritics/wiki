# MaterializeInc/materialize

> A streaming database that incrementally maintains SQL views over changing data, and serves them with strong consistency over the PostgreSQL protocol.

[GitHub repo](https://github.com/MaterializeInc/materialize) ·
[Official website](https://materialize.com) ·
License: BUSL-1.1 (converts to Apache-2.0 after 4 years)

## Overview

Materialize is an operational data store built around one idea: instead of
re-running a query each time you ask, keep its answer continuously up to date as
the input changes. You write a `CREATE MATERIALIZED VIEW` in PostgreSQL-dialect
SQL, point it at change streams (Postgres/MySQL/SQL Server CDC, or Kafka), and
Materialize maintains the result incrementally — applying only the deltas implied
by each insert, update, or delete rather than recomputing from scratch[^1].

The engineering that makes this tractable is Timely Dataflow and Differential
Dataflow, the incremental-computation frameworks written by Frank McSherry, a
Materialize co-founder[^2]. Differential dataflow lets a multi-way join or nested
aggregation react to a single upstream row change in time roughly proportional to
the change, not to the size of the data. This is the same class of problem Flink
and ksqlDB address, but Materialize's defining claim is *strong consistency*:
every answer is the correct result over some recent, specific version of the
inputs — never approximate or eventually-consistent, even across joins of data
from multiple upstream systems[^3].

The tension to understand before adopting is that this guarantee is bought with
state. Incrementally maintained views hold their intermediate arrangements in
memory (with disk spill in newer versions), so Materialize has historically been
RAM-bound and priced accordingly. It is a serving and integration layer over your
source-of-truth databases — not a replacement OLTP store, nor a cheap warehouse.

## Getting Started

The community edition ("emulator") ships as a single-node Docker image, free for
deployments under 24 GiB memory and 48 GiB disk[^4]:

```bash
docker run -d -p 6875:6875 --name mz materialize/materialized:latest
# 6875 is Materialize's default pgwire port
psql -U materialize -h localhost -p 6875 materialize
```

```sql
-- A built-in load generator gives you a changing source with no setup.
CREATE SOURCE auction FROM LOAD GENERATOR AUCTION FOR ALL TABLES;

-- MATERIALIZED means: maintain this result eagerly and durably as data changes.
CREATE MATERIALIZED VIEW hot_items AS
  SELECT a.item, count(*) AS bids
  FROM bids b JOIN auctions a ON b.auction_id = a.id
  GROUP BY a.item;

-- An index keeps results in memory for fast point lookups.
CREATE INDEX ON hot_items (item);

-- Read a correct, current answer immediately — or stream changes as they land.
SELECT * FROM hot_items ORDER BY bids DESC LIMIT 10;
SUBSCRIBE (SELECT * FROM hot_items);   -- push-based change feed
```

## Architecture / How It Works

Materialize is written in Rust and, since its "platform" rewrite, is a
cloud-native system rather than a single binary. Two planes matter:

- **Storage** — a component called `persist` writes source data and materialized
  results as immutable batches to cloud object storage (e.g. S3), with a separate
  durable consensus/metadata store (historically CockroachDB) holding the catalog
  and timestamp oracle. This decouples durability from any running compute[^5].
- **Compute** — SQL dataflows run inside user-defined **clusters**, each with one
  or more **replicas** (worker processes executing the same dataflow). Multiple
  replicas give high availability via *multi-active replication*: all replicas
  compute, and reads come from whichever is ready. Clusters scale horizontally.

The control-plane process (`environmentd`) plans SQL and coordinates timestamps;
`clusterd` processes do the dataflow work. Every datum carries a logical
timestamp, and the timestamp oracle is what enables strict consistency: a
`SELECT` is answered at a timestamp reflecting a consistent cut across all
sources feeding the view. The optimizer performs subquery decorrelation
automatically, and **delta joins** let a multi-way join (tested up to 64
relations) avoid the intermediate-state blow-up of binary-join plans[^1].
Overlapping subplans across views share the same underlying arrangements, so
layered views do not duplicate state.

## Production Notes

**Memory is the resource that matters.** Each index and materialized view keeps
its arrangement resident so it can react to deltas. A view over high-cardinality
keys, or a join with large intermediate state, costs RAM regardless of how small
the final result is. Newer versions spill arrangements to local disk, but sizing
clusters by working-set memory — not result size — is the correct mental model,
and it is the dominant cost driver on the cloud's credit-based pricing.

**Source replication is a live coupling.** A Postgres source consumes a
replication slot; if Materialize falls behind or is paused, the upstream WAL is
retained and can fill the primary's disk. MySQL and SQL Server sources have
analogous binlog/CDC retention concerns. Treat the slot/CDC configuration as
monitored production infrastructure, not a one-time setup.

**It is not OLTP and not a warehouse.** Materialize serves and maintains views; it
is not built for high-throughput transactional point writes, nor is it a cheap
store for ad-hoc scans over cold history. The sweet spot is a bounded set of
known, expensive queries you need fresh at all times.

**PostgreSQL compatibility is a large subset, not parity.** The wire protocol and
dialect mean existing psql/drivers/BI tools and dbt Core work unchanged, but some
Postgres functions and features are unimplemented; check before assuming a
built-in exists. And the single-node Docker emulator understates ops: a
production self-managed deployment is a Kubernetes install (operator + object
storage + metadata store), closer to running a distributed system.

## When to Use / When Not

**Use when:**
- You have expensive read queries (complex joins, aggregations) that must always
  be fresh, expressed in SQL rather than a hand-written pipeline.
- You need CQRS read-offload, a CDC integration hub, or real-time data products
  fed from Postgres/MySQL/Kafka — with correctness you can't get from
  eventually-consistent streaming systems.
- You feed fresh, consistent context into dashboards, services, or AI/RAG
  pipelines via ordinary Postgres drivers.

**Avoid when:**
- Your workload is transactional (many small writes, point updates) — use the
  source database directly.
- You need cheap, occasional scans over large cold datasets — a warehouse
  (Snowflake, BigQuery, ClickHouse) is far cheaper per byte.
- Your queries are one-off and unpredictable, or view state won't fit your memory
  budget — incremental maintenance pays off for stable, repeated queries.

## Alternatives

- risingwavelabs/risingwave — the closest direct competitor: a Postgres-compatible
  streaming database, also in Rust, with more emphasis on cheap object-storage
  state. Use when you want a similar model with a different cost/state profile.
- apache/flink — lower-level, general stream processing. Use when you need
  arbitrary streaming logic and are willing to own more operational complexity and
  weaker out-of-the-box consistency guarantees.
- confluentinc/ksql — Kafka-native streaming SQL. Use when your world is entirely
  Kafka and eventual consistency is acceptable.
- readysettech/readyset — a dataflow-based (Noria-lineage) incremental read cache
  for Postgres/MySQL. Use when you want a mostly-transparent query cache rather than
  a full SQL serving layer.
- ClickHouse/clickhouse — columnar OLAP with incremental materialized views. Use
  when you want fast analytics over large data and can accept eventual freshness.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-02 | Company founded (Arjun Narayan, Frank McSherry); repo created[^2]. |
| 0.1 | 2020-02 | First public release: single-binary engine, source-available under BSL 1.1[^6]. |
| platform | 2022–2023 | Rewrite separating storage (`persist`/S3) from compute (clusters/replicas)[^5]. |
| Cloud GA | 2023 | Fully managed cloud service reaches general availability; weekly release cadence[^3]. |

## References

[^1]: Materialize README and docs — SQL support, delta joins, incremental
maintenance. https://github.com/MaterializeInc/materialize
[^2]: Timely Dataflow and Differential Dataflow, by Frank McSherry (Materialize
co-founder). https://github.com/TimelyDataflow/differential-dataflow
[^3]: Materialize blog, "Strong consistency in Materialize."
https://materialize.com/blog/strong-consistency-in-materialize/
[^4]: Materialize download / community edition (free under 24 GiB memory, 48 GiB
disk). https://materialize.com/download/
[^5]: Materialize documentation — architecture (storage, compute, clusters,
replicas). https://materialize.com/docs/
[^6]: Materialize LICENSE — Business Source License 1.1, converting to Apache 2.0
after 4 years. https://github.com/MaterializeInc/materialize/blob/main/LICENSE

## Tags

rust, streaming-database, sql, materialized-views, incremental-computation, change-data-capture, stream-processing, postgresql-protocol, differential-dataflow, real-time-analytics, kafka, operational-data-store
