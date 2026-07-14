# debezium/debezium

> Log-based change data capture for a range of databases, streamed as ordered events — usually through Kafka Connect.

[GitHub repo](https://github.com/debezium/debezium) ·
[Official website](https://debezium.io) ·
[License: Apache-2.0](https://github.com/debezium/debezium/blob/main/LICENSE.txt)

## Overview

Debezium is a change data capture (CDC) platform: it reads a database's own
transaction log — MySQL binlog, PostgreSQL WAL, MongoDB oplog/change streams,
Oracle redo logs, SQL Server CDC tables — and emits one event per committed
row-level change. It started at Red Hat around 2016 and reached 1.0 in
December 2019[^1]. Only committed changes surface, events are totally ordered
per table, and a consumer that stops can resume exactly where it left off.
The pitch is that you get a single, uniform event model across heterogeneous
databases instead of hand-writing triggers or polling.

The defining architectural bet is Kafka Connect. A Debezium connector is
normally deployed as a Kafka Connect source task: Kafka provides the
durability, ordering, replication, and offset tracking, and Debezium provides
the log-reading logic per database. This is why Debezium is reliable out of
the box, and also why "just adopt Debezium" often means "also operate a Kafka
cluster." The project offers two escape hatches — the *embedded engine* (run a
connector as a library inside your own JVM process) and *Debezium Server* (a
standalone runtime that sinks to Kinesis, Google Pub/Sub, Pulsar, Redis
Streams, and others) — but the Kafka path is the mature, best-tested one.

The core tension is that log-based CDC is invasive at the database layer.
Debezium needs replication privileges, a logical replication slot or CDC
feature enabled, and retained log segments. When it falls behind or a slot is
left open, the *source database* pays the price — not just the pipeline.

## Getting Started

Debezium connectors are configured as JSON registered against a running Kafka
Connect cluster. A minimal PostgreSQL connector:

```bash
# Register a connector with a Kafka Connect worker (REST API on :8083)
curl -X POST http://localhost:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "inventory-postgres",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "db",
      "database.port": "5432",
      "database.user": "debezium",
      "database.password": "secret",
      "database.dbname": "inventory",
      "topic.prefix": "inv",
      "plugin.name": "pgoutput",
      "table.include.list": "public.customers"
    }
  }'
```

Each change to `public.customers` lands on topic `inv.public.customers` as an
envelope with `before`, `after`, `op` (`c`/`u`/`d`/`r`), and `source` metadata.
Most consumers apply the `ExtractNewRecordState` SMT (formerly "unwrap") to
flatten the envelope into a plain row image. To embed instead of running
Kafka Connect, add `debezium-embedded` and drive `DebeziumEngine` directly.

## Architecture / How It Works

A connector's life has two phases. On first start it takes a **snapshot**:
a consistent read of existing table contents emitted as synthetic `r` (read)
events, so consumers see current state before streaming begins. It then
**streams** by tailing the log from the snapshot's offset onward. Offsets and,
for relational connectors, a **schema history** are persisted (Kafka topics in
distributed mode) so restarts resume without re-snapshotting.

Per-database mechanics differ sharply:

- **PostgreSQL** — uses logical decoding via a replication slot; `pgoutput`
  (built in) is now the standard plugin, with `decoderbufs`/`wal2json` as
  older options. The slot is the main operational hazard (see below).
- **MySQL/MariaDB** — reads the binlog; requires `ROW` binlog format and
  appropriate replication grants. Schema changes are tracked via the schema
  history topic.
- **MongoDB** — consumes change streams (oplog on older deployments).
- **Oracle** — LogMiner by default, or XStream (which needs a GoldenGate
  license); historically the most operationally involved connector.
- **SQL Server** — reads the tables populated by SQL Server's own CDC feature,
  which must be enabled per table.

**Incremental snapshots** (introduced in 1.6, based on the DBLog/watermark
algorithm) let you snapshot new tables while streaming continues, chunk by
chunk, driven through a **signaling** table or Kafka topic — a major
improvement over the original stop-the-world snapshot lock[^2]. Debezium also
ships Single Message Transforms for the **outbox pattern** (transactional
outbox event routing), content-based routing, and message filtering, plus a
**JDBC sink connector** for the ingest side.

## Production Notes

The most common outage is a **PostgreSQL replication slot that stops
advancing**. If the connector is down, paused, or lagging, Postgres retains
WAL for the slot and disk fills — potentially taking the primary offline.
Monitor `pg_replication_slots.confirmed_flush_lsn` and alert on WAL growth;
never leave an orphaned slot behind after removing a connector.

**One connector monitors one database server.** There is no built-in
sharding of a single server's log across tasks — a Debezium source task is
effectively single-threaded per server, so very high write volume is bounded
by how fast one task decodes the log, not by adding Connect workers.

**Schema history is load-bearing** for relational connectors. Losing or
corrupting the schema history topic (or the MySQL binlog it was built from)
can leave a connector unable to interpret events; the recovery path is often a
re-snapshot. Treat that topic with the same care as consumer offsets.

**Snapshots on large tables** are I/O- and time-intensive and, in the classic
mode, held read locks. Use incremental snapshots and tune chunk size; budget
for the snapshot to run for hours on multi-hundred-GB tables.

**Oracle LogMiner** can impose meaningful load on the source and lag under
heavy redo volume; sizing and archive-log retention need real attention.
Across all connectors, **binlog/WAL/redo retention** must exceed worst-case
connector downtime or you lose the ability to resume and must re-snapshot.

Delivery is **at-least-once** by default: duplicates are possible after
restarts, so downstream consumers should be idempotent on the event key.
Version upgrades occasionally change topic naming, default SMT names
(`unwrap` → `ExtractNewRecordState`), or configuration keys (the connector
config overhaul in 2.0 renamed several properties), so read release notes
before jumping majors.

## When to Use / When Not

**Use when:**
- You need reliable, ordered, low-latency propagation of DB changes to caches,
  search indexes, data warehouses, or other services.
- You want to eliminate dual-writes / the transactional outbox is a fit.
- You already run — or are willing to run — Kafka and Kafka Connect.
- You need one uniform change model across several database engines.

**Avoid when:**
- You can't grant replication access or enable logical decoding / CDC on the
  source database.
- You want a zero-infrastructure managed sync and aren't willing to operate
  Kafka (consider a hosted CDC service instead).
- Your need is periodic bulk ETL, not per-row streaming — a batch tool is
  simpler.
- Ultra-high single-server write throughput exceeds what one decoding task can
  keep up with.

## Alternatives

- airbyte/airbyte — use when you want managed, connector-catalog ELT with a UI
  and don't want to run Kafka (Airbyte itself embeds Debezium for CDC sources).
- ververica/flink-cdc — use when your sink is Apache Flink and you want CDC
  ingestion inside a stream-processing job rather than through Kafka Connect.
- Confluent/Fivetran/AWS DMS (proprietary) — use when you want a fully managed
  CDC service and can accept vendor lock-in and cost over self-hosting.
- Maxwell / mysql binlog tailers — use when you only need MySQL CDC and want a
  smaller, single-database footprint than Debezium's multi-engine platform.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2016–2019 | Early releases; MySQL first, then Postgres/Mongo/others. |
| 1.0.0.Final | 2019-12 | First GA; stable core model[^1]. |
| 1.6 | 2021 | Incremental snapshots + signaling introduced[^2]. |
| 2.0.0.Final | 2022-10 | Major line; connector config and internals overhaul. |
| 3.0.0.Final | 2024-10 | Modern JDK/Kafka baseline; continued connector growth. |

## References

[^1]: Debezium blog / releases — 1.0 GA announcement (December 2019). https://debezium.io/blog/
[^2]: Debezium documentation, "Incremental snapshots" and signaling. https://debezium.io/documentation/reference/stable/configuration/signalling.html

## Tags

java, cdc, change-data-capture, kafka, kafka-connect, streaming, database, postgresql, mysql, mongodb, data-pipeline, event-streaming
