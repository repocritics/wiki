# dolthub/dolt

> A MySQL-compatible SQL database with Git-style version control built into the storage engine — you can branch, diff, and merge tables the way Git does files.

[GitHub repo](https://github.com/dolthub/dolt) ·
[Official website](https://www.dolthub.com) ·
[License: Apache-2.0](https://github.com/dolthub/dolt/blob/main/LICENSE)

## Overview

Dolt is a relational database that treats version control as a first-class storage feature rather than an application-level convention. It speaks the MySQL wire protocol and dialect, so existing MySQL clients, drivers, and ORMs connect to it unchanged, while every table carries a full commit history you can branch, diff, merge, push, and pull. The project's own summary — "Git versions files, Dolt versions tables" — is accurate: the CLI mirrors Git command-for-command (`dolt add`, `dolt commit`, `dolt merge`, `dolt push`), and the same operations are reachable in SQL through system tables and stored procedures[^1].

It is built by DoltHub (the company), which also runs a hosted data-sharing site of the same name, a self-hostable server product (DoltLab), and a managed offering (Hosted Dolt). A Postgres-dialect sibling, [dolthub/doltgresql](https://github.com/dolthub/doltgresql), reuses the same versioned storage under a Postgres wire protocol and is in beta[^2]. The repository has recently leaned into an "agent memory" positioning — versioned, mergeable state for multi-agent systems — reflected in its GitHub topics, though the engine itself is a general-purpose OLTP database.

The defining tension is cost-of-history. Storing every version of every row, and keeping diff and three-way merge cheap, requires a content-addressed storage engine (prolly trees) that behaves differently from InnoDB's B-trees. That buys structural sharing and commit-sized diffs, but it also means Dolt is not a drop-in performance replacement for MySQL: it historically carried a measurable latency and storage overhead, which the team tracks publicly and has narrowed over time[^3].

## Getting Started

```bash
# macOS
brew install dolt
# Linux / macOS (script)
sudo bash -c 'curl -L https://github.com/dolthub/dolt/releases/latest/download/install.sh | bash'
```

Dolt ships as a single self-contained binary (~103 MB) with both a server and a MySQL-compatible client built in.

```bash
mkdir getting_started && cd getting_started
dolt sql-server &                 # MySQL-compatible server on :3306
dolt -u root -p "" sql            # or connect any MySQL client to 127.0.0.1:3306
```

```sql
CREATE DATABASE getting_started;
USE getting_started;
CREATE TABLE employees (id INT PRIMARY KEY, last_name VARCHAR(255), first_name VARCHAR(255));

-- version control lives in SQL: stored procedures + system tables
CALL dolt_add('employees');
CALL dolt_commit('-m', 'Created initial schema');

INSERT INTO employees VALUES (0, 'Sehn', 'Tim');
SELECT * FROM dolt_status;        -- working-set changes
SELECT * FROM dolt_diff_employees;-- row-level diff vs last commit
CALL dolt_commit('-am', 'Add first employee');
SELECT * FROM dolt_log;           -- commit history
```

The `dolt_<command>` naming is consistent: every CLI verb has a `dolt_<verb>` procedure, and reads (log, diff, status, blame, history) surface as `dolt_*` system tables that participate in ordinary SQL queries.

## Architecture / How It Works

**Storage engine (prolly trees).** Dolt's core data structure is a probabilistic B-tree — a "prolly tree" — a content-addressed, Merkle-style structure whose node boundaries are determined by a rolling hash of the content rather than by insertion order[^4]. Two consequences fall out of this: identical subtrees are shared across commits (so history is deduplicated), and a diff between any two commits costs work proportional to the size of the change, not the size of the table. Three-way merge is a structural operation over these trees, which is why Dolt can merge branches at the row level with conflict detection rather than replaying a log. The storage layer descends from Attic Labs' Noms; Dolt later moved to its own on-disk format and provides `dolt migrate` to convert legacy databases[^4].

**SQL engine.** The query layer is [dolthub/go-mysql-server](https://github.com/dolthub/go-mysql-server), a pure-Go SQL engine, with a query parser derived from Vitess's `sqlparser`. This is why a fresh server reports a version string like `5.7.9-Vitess` on connect. Dolt supports foreign keys, secondary indexes, triggers, check constraints, stored procedures, and multi-table joins — it is a genuine relational engine, not a key-value store with a SQL veneer.

**Version control as SQL.** Branches are cheap pointers into the commit graph. Writes happen against a working set; `dolt_commit` snapshots it. Merges, cherry-picks, resets, and conflict resolution are all exposed both on the CLI and as procedures, so an application can drive branching logic transactionally. A Dolt commit is distinct from a SQL transaction `COMMIT`; the `@@dolt_transaction_commit` system variable can bind the two so every transaction produces a Dolt commit.

**cgo dependency.** The build links C code (via cgo), so building from source needs a working C toolchain in addition to Go (`go install ./cmd/dolt` from the `go/` directory).

## Production Notes

**Performance overhead vs MySQL.** Prolly-tree storage is the price of cheap history. DoltHub publishes an ongoing sysbench comparison against MySQL and has stated a goal of staying within a small multiple of MySQL latency; historically the gap was several times slower on some workloads and has narrowed substantially, but you should benchmark your own read/write mix rather than assume parity[^3]. Point lookups and range scans over content-addressed trees have different cache behavior than InnoDB.

**Storage growth and garbage collection.** Keeping full history means the database grows with churn, not just with live row count. `dolt gc` reclaims unreferenced chunks, but it is a deliberate operation — schedule it; it is not automatic the way InnoDB purge is. Large historical rewrites (`dolt filter-branch`) and long-lived high-write databases are the cases where storage and GC planning matter most.

**MySQL compatibility is high but not total.** The wire protocol and dialect coverage are broad, yet edge cases in collations, information_schema details, and specific function/behavior corners differ from MySQL. Test your ORM's migrations and any raw SQL that depends on MySQL internals. Notably, MySQL 9.0's default `caching_sha2_password` authentication does not connect to a Dolt server out of the box — the project recommends MySQL 8.4 (LTS) clients or explicit auth configuration[^5].

**Merge conflicts are real conflicts.** Row- and schema-level conflicts surface through the `dolt_conflicts_<table>` system tables and must be resolved before commit, exactly like Git. Applications that branch-per-request or branch-per-agent need a conflict-resolution strategy; concurrent schema changes on divergent branches are the sharpest edge.

**Binary size and single-binary ops.** The ~103 MB static binary simplifies deployment (no separate server package), but it is large for slim containers; use the official `dolthub/dolt` and `dolthub/dolt-sql-server` images if you want a maintained baseline.

## When to Use / When Not

**Use when:**
- You need auditable, time-travelable data with row-level lineage (`dolt_blame`, `dolt_history_*`) — regulated data, reference datasets, config-as-data.
- You want to branch and merge datasets: collaborative data curation, staging/QA data that diffs against production, or per-agent scratch state you later merge.
- You want to distribute and collaborate on a dataset like a repo (clone/fork/push/pull) rather than shipping dumps.
- You already speak MySQL and want version control without changing your driver.

**Avoid when:**
- You need MySQL-level throughput at scale today and cannot absorb any storage/latency overhead — run MySQL/Postgres and version at the application layer.
- Your data churns enormously and history has no value — you'd pay for lineage you throw away.
- You depend on MySQL replication topology, specific storage-engine features, or 100% behavioral parity with a particular MySQL version.
- You want Postgres — track Doltgres, but note it is still beta.

## Alternatives

- treeverse/lakeFS — Git-like branching/merging over object-storage data lakes (Parquet/S3), not a live SQL database. Use when your data is files in object storage, not OLTP tables.
- iterative/dvc — version control for ML datasets and model files layered on Git. Use when you version large files/pipelines alongside code, not queryable rows.
- terminusdb/terminusdb — a versioned graph/document database with branch-and-merge. Use when your model is graph/document rather than relational MySQL.
- vitessio/vitess — MySQL sharding with schema-branch workflows (the PlanetScale model). Use when you want horizontal MySQL scale and schema branching, not full data version control.
- dolthub/doltgresql — the same versioned engine with a Postgres dialect. Use when you need Postgres compatibility and can accept beta maturity.

## History

| Version | Date | Notes |
|---------|------|-------|
| open source | 2019-08 | Repo public; "Git for Data" CLI, storage descended from Noms[^1][^4]. |
| new storage format | ~2022 | Move off legacy Noms format to Dolt's own format; `dolt migrate` added[^4]. |
| 1.0 | 2023-05 | Stability milestone — format/compatibility guarantees for production use[^6]. |
| Doltgres beta | 2024–2025 | Postgres-dialect sibling reaches beta[^2]. |

## References

[^1]: Dolt README — "Dolt is Git for Data." https://github.com/dolthub/dolt
[^2]: Doltgres (Postgres flavor of Dolt). https://github.com/dolthub/doltgresql
[^3]: DoltHub, "Dolt vs. MySQL benchmarks" (ongoing sysbench comparison and latency-multiple tracking). https://docs.dolthub.com/sql-reference/benchmarks/latency
[^4]: DoltHub, "How Dolt Stores Table Data" / prolly trees and the Noms lineage. https://www.dolthub.com/blog/2020-04-01-how-dolt-stores-table-data/
[^5]: Dolt README / DoltHub, MySQL 9.0 `caching_sha2_password` compatibility note. https://www.dolthub.com/blog/2024-12-11-mysql9-and-caching-sha2-auth-support/
[^6]: DoltHub, "Dolt 1.0" release announcement. https://www.dolthub.com/blog/2023-05-03-dolt-1-dot-0/

## Tags

golang, database, mysql, sql, version-control, git-for-data, prolly-tree, data-versioning, oltp, merge, agent-memory
