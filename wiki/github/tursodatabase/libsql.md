# tursodatabase/libsql

> An open-contribution fork of SQLite that adds a network server, embedded replicas, and extensions — while inheriting SQLite's single-writer core.

[GitHub repo](https://github.com/tursodatabase/libsql) ·
[Official website](https://turso.tech/libsql) ·
[License: MIT](https://github.com/tursodatabase/libsql/blob/main/LICENSE.md)

## Overview

libSQL is a fork of SQLite created and maintained by Turso (formerly ChiselStrike), first published in 2022[^1]. SQLite is public domain and technically excellent, but it famously does not accept external contributions and ships no code of conduct. libSQL's premise is social before it is technical: take the SQLite codebase, relicense the fork under MIT, and open it to community contributions[^1]. On top of that it layers features SQLite upstream would never merge — a network server, embedded replicas, WebAssembly user-defined functions, encryption at rest, and `ALTER TABLE` column-type changes[^3].

The defining tension in 2026 is that Turso has redirected new feature work to a different project. The team is rewriting a SQLite-compatible engine from scratch in Rust — [tursodatabase/turso](https://github.com/tursodatabase/turso), previously "Limbo" — which is *not* a fork and targets things a fork cannot reach, notably concurrent writes and bidirectional offline sync[^2]. libSQL remains actively maintained (commits land regularly, and the codebase powers Turso's hosted platform), but the maintainers themselves now tell new projects to evaluate the Rust rewrite first[^2]. Reading this page as a critic: libSQL is production-usable and stable, but you are adopting the *previous* generation of its authors' roadmap.

Because it is a fork of the C core, libSQL inherits SQLite's fundamental architecture — the bytecode VM, the B-tree pager, the WAL — and therefore its fundamental limits. The single-writer model is the one that matters most operationally, and it is exactly the constraint the Rust rewrite exists to break.

## Getting Started

Build the SQLite-compatible C library and shell from source (a Rust toolchain is required even though the core is C):

```sh
git clone https://github.com/tursodatabase/libsql
cd libsql
cargo xtask build
```

Most users consume libSQL through a driver rather than building it. The Rust client (`libsql` crate) is the reference API:

```rust
use libsql::Builder;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Local embedded file — behaves like plain SQLite.
    let db = Builder::new_local("local.db").build().await?;
    let conn = db.connect()?;

    conn.execute("CREATE TABLE users (id INTEGER, name TEXT)", ()).await?;
    conn.execute("INSERT INTO users VALUES (1, ?1)", ["Tom"]).await?;

    let mut rows = conn.query("SELECT name FROM users", ()).await?;
    while let Some(row) = rows.next().await? {
        println!("{}", row.get_str(0)?);
    }
    Ok(())
}
```

An embedded replica keeps a local file synced from a remote primary — reads hit the local copy, writes are forwarded to the primary:

```rust
let db = Builder::new_remote_replica("local.db", url, token).build().await?;
db.sync().await?;              // pull latest frames from the primary
let conn = db.connect()?;      // reads are now local and fast
```

Official drivers exist for TypeScript/JS, Rust, and Go; Python and C are experimental[^2].

## Architecture / How It Works

**The core** (`libsql-sqlite3`) is the SQLite C source with patches. Query parsing, the bytecode virtual machine, the B-tree, the pager, and the WAL are all inherited. libSQL commits to reading and writing the standard SQLite file format unless you opt into format-changing features like encryption[^3].

**Extensions on the core** include a virtual WAL interface (the hook that makes replication possible without touching the pager), WebAssembly UDFs, randomized ROWID, and `ALTER TABLE` support for changing column types and constraints[^3].

**libsql-server** (historically `sqld`) is a Rust process that wraps the C engine and exposes it over the network via the Hrana protocol — SQLite queries carried over HTTP and WebSocket, so remote clients can talk to a database the way they would to PostgreSQL or MySQL[^5]. This is the piece SQLite proper has no answer to.

**Embedded replicas** are the headline feature. A client holds a full local SQLite file and periodically pulls new WAL frames from a remote primary. Reads are served locally with zero network latency; writes are shipped to the primary and applied there, then flow back on the next sync[^4]. The consistency model is therefore *not* strong by default: a local replica can be stale between syncs, and read-your-writes requires enabling the corresponding option so reads wait for the write to be reflected.

The write path is where SQLite's DNA shows: **all writes serialize through a single writer.** Replicas fan out reads, but they do not add write concurrency — they forward to one primary. Scaling writes is not a tuning problem here; it is an architectural ceiling.

## Production Notes

**Single-writer means write latency is dominated by the primary.** With embedded replicas, every write is a round trip to wherever the primary lives. If your primary is in one region and your app in another, write latency is the inter-region RTT, not local disk time. Reads are local and fast; writes are not. Design around read-heavy workloads.

**Sync is explicit and eventually consistent.** `db.sync()` (or a periodic sync interval) pulls frames; between syncs the replica is stale. Code that writes then immediately reads the same row from a replica can see the old value unless read-your-writes is enabled. This is the most common correctness surprise for teams treating a replica like a normal local database.

**Encryption at rest changes the file format.** Turn it on and the file is no longer a standard SQLite file; the compatibility guarantee only holds for the unencrypted path[^3]. Budget for that in any tooling that expects to open the file with vanilla SQLite.

**Driver maturity is uneven.** Rust and TypeScript/JS are the well-trodden paths; Go ships both a CGO driver and a pure-Go HTTP client; Python and C are labelled experimental[^2]. Pick a language where the driver is first-class, not one where you are the driver's first serious user.

**Roadmap risk is the real caveat.** New capabilities (concurrent writes, offline sync) are being built in the separate Rust project, not here[^2]. libSQL gets maintenance and security attention, and it is battle-tested under Turso's hosted platform, but if you need a feature that does not exist yet, it will likely land in Turso database first — and Turso is still beta. The open-issue count (400+) is consistent with a widely-used project in a maintenance-forward phase rather than one under heavy new-feature churn.

## When to Use / When Not

**Use when:**
- You want SQLite semantics plus a network server or read replicas without bolting on a separate system.
- Your workload is read-heavy and can tolerate eventually-consistent local replicas.
- You are already on Turso's hosted platform, or want a self-hostable engine with that on-ramp.
- You value that the fork is MIT-licensed and accepts contributions, unlike upstream SQLite.

**Avoid when:**
- You need concurrent writers or high write throughput — the single-writer ceiling is inherited and unmovable here.
- You are starting greenfield and can evaluate the Rust rewrite; the maintainers point new projects there[^2].
- You need strong read-your-writes consistency across replicas without extra care.
- Plain embedded SQLite already covers you — a single-process local app gains nothing from the fork.

## Alternatives

- tursodatabase/turso — the same team's from-scratch Rust rewrite; use it when starting fresh and you want concurrent writes or offline sync and can tolerate beta status.
- sqlite/sqlite — upstream SQLite; use it when you need only a local embedded engine, want the canonical public-domain code, and don't need a server or replicas.
- rqlite/rqlite — distributed SQLite over Raft; use it when you need a strongly-consistent, highly-available cluster rather than eventually-consistent replicas.
- vlcn-io/cr-sqlite — CRDT extension for multi-writer, conflict-free sync; use it when many nodes must write and merge without a single primary.
- pocketbase/pocketbase — SQLite-backed application backend with auth and REST; use it when you want a batteries-included backend, not just the database engine.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repo published | 2022-09 | libSQL fork of SQLite opened by ChiselStrike/Turso[^1]. |
| Server + Hrana | 2023 | `sqld`/libsql-server exposes SQLite over HTTP/WebSocket[^5]. |
| Embedded replicas | 2023–2024 | Local replica synced from a remote primary[^4]. |
| Encryption at rest | 2024 | Optional, format-changing[^3]. |
| Rust rewrite pivot | 2025 | New features move to tursodatabase/turso (not a fork)[^2]. |
| Actively maintained | 2026-07 | Last push 2026-07-12; maintenance + platform use continues[^2]. |

## References

[^1]: libSQL manifesto / product vision — Turso. https://turso.tech/libsql-manifesto
[^2]: libSQL README, "Turso database and libSQL are two different projects" note. https://github.com/tursodatabase/libsql
[^3]: libSQL extensions documentation. https://github.com/tursodatabase/libsql/blob/main/libsql-sqlite3/doc/libsql_extensions.md
[^4]: Turso documentation — embedded replicas. https://docs.turso.tech
[^5]: Hrana protocol and libsql-server. https://github.com/tursodatabase/libsql/tree/main/docs

## Tags

database, sqlite, embedded-database, fork, c, rust, replication, edge-database, webassembly, single-writer
