# automerge/automerge

> A JSON-like CRDT with a compact binary history format and a sync protocol —
> "PostgreSQL for your local-first app."

[GitHub repo](https://github.com/automerge/automerge) ·
[Official website](https://automerge.org) ·
[License: MIT](https://github.com/automerge/automerge/blob/main/LICENSE)

## Overview

Automerge is a Conflict-free Replicated Data Type (CRDT) library out of the
Ink & Switch research lab, closely associated with Martin Kleppmann and the
2019 "local-first software" essay that named the movement[^1]. It gives you a
JSON-shaped document (maps, lists, text, counters) that multiple devices can
modify independently — offline included — and merge deterministically without
a central server deciding conflicts. The stated ambition is to be the
persistence layer for local-first apps the way relational databases are for
server apps.

This repository is the *second* Automerge: a Rust core (formerly
`automerge-rs`, created December 2019) compiled to WebAssembly and wrapped by
an idiomatic JavaScript package, with a C FFI for other language bindings. It
became canonical with Automerge 2.0 in early 2023[^2]; the original
pure-JavaScript implementation — which had well-known performance and memory
problems — lives on as `automerge/automerge-classic`. Automerge 3 (2025) cut
memory usage roughly 10×[^3], addressing the longest-running complaint.

The defining tradeoff: Automerge keeps the **full operation history** of every
document. That buys reliable merges of arbitrarily old offline edits,
attribution, and time-travel — and costs you monotonically growing documents
with no way to discard history short of forking a fresh document. At ~6.4k
stars with two full-time Ink & Switch-funded maintainers and pushes within the
last week, it is an actively developed research-lab project rather than a
big-vendor product; the Rust API layer is still explicitly underdocumented[^4].

## Getting Started

```bash
npm install @automerge/automerge
```

```js
import * as Automerge from "@automerge/automerge";

// Device A creates a document
let doc1 = Automerge.init();
doc1 = Automerge.change(doc1, "Add card", d => {
  d.cards = [{ title: "Rewrite everything", done: false }];
});

// Device B gets a copy, both edit concurrently
let doc2 = Automerge.merge(Automerge.init(), doc1);
doc1 = Automerge.change(doc1, d => { d.cards[0].done = true; });
doc2 = Automerge.change(doc2, d => { d.cards.push({ title: "Ship it", done: false }); });

// Merge converges — both edits survive, no conflict handler needed
const merged = Automerge.merge(doc1, doc2);
const bytes = Automerge.save(merged);   // compact binary, persist anywhere
```

For real applications, start with `automerge-repo` (same org), which adds the
storage adapters, network adapters, and document-handle lifecycle the core
library deliberately does not include[^5].

## Architecture / How It Works

Automerge is operation-based: every mutation becomes an operation stamped with
an actor ID and sequence number, and a document is the deterministic
materialization of its full operation DAG. Any two replicas that have seen the
same set of changes render identical state, regardless of delivery order.

- **Rust core, WASM/FFI shell.** All CRDT logic lives in the Rust crate. The
  JavaScript package calls into `automerge-wasm` and exposes a functional API
  — documents are frozen; `Automerge.change()` returns a new document plus a
  serialized change. `automerge-c` feeds Swift and other bindings. The Rust
  crate is usable directly but low-level; `autosurgeon` layers a derive-macro
  API over it for Rust applications[^4].
- **Columnar binary format.** History is column-compressed (op types together,
  actor IDs together), which is why documents with hundreds of thousands of
  edits serialize to sizes close to the raw text. The format is publicly
  specified[^6] — a deliberate bet on Automerge-the-format outliving
  Automerge-the-library.
- **Sync protocol.** A per-peer, transport-agnostic protocol exchanges
  document heads and Bloom filters of change hashes to work out which changes
  the other side is missing, transmitting only those; automerge-repo ships
  WebSocket, BroadcastChannel, and other adapters.
- **Text and rich text.** Collaborative text is a first-class sequence CRDT
  with character-level merging, plus a marks API for rich-text formatting
  spans following Ink & Switch's Peritext design[^7].

## Production Notes

- **History is forever.** No compaction discards operations. Heavily edited
  documents grow without bound, and loading one materializes it fully in
  memory — no partial loading, no query engine. Document granularity is your
  sharding decision: many small documents beat one giant one.
- **Memory was the historical footgun.** Pre-3.0, multi-megabyte documents
  could consume hundreds of megabytes of RAM when loaded. Automerge 3's ~10×
  reduction[^3] changes the calculus, but budget load-time memory well above
  on-disk size.
- **Actor ID discipline.** An actor ID must never be used concurrently from
  two places. Reusing one (copied local storage, two tabs sharing an ID)
  produces corrupt histories that are painful to diagnose.
- **The bare library does nothing about persistence or networking.** Teams
  wiring `save()`/`merge()` by hand rediscover automerge-repo's problems
  (incremental saves, per-peer sync-state persistence, handle lifecycle) one
  incident at a time. Use automerge-repo unless you have a reason not to[^5].
- **WASM deployment friction.** The core is a WebAssembly blob — meaningful
  bundle weight, and bundler WASM handling (Vite/Webpack, Node vs browser
  targets) is a recurring source of setup issues.
- **CRDT semantics ≠ your semantics.** Merges are deterministic, not
  necessarily what the user meant: concurrent list edits interleave, counters
  need the dedicated Counter type, and cross-field invariants ("these must sum
  to 100") cannot be enforced by the merge. Application-level validation stays
  your job.
- **Ecosystem is thinner than Yjs.** ProseMirror/CodeMirror bindings exist,
  but the third-party ecosystem and answer surface are smaller.

## When to Use / When Not

**Use when:**
- You are building local-first or offline-first apps where devices edit
  independently for long periods and must merge cleanly.
- You want full history: attribution, audit, time-travel, or syncing a peer
  that has been offline for months.
- You need one CRDT core across web (WASM), Rust, Swift, and C hosts.
- Document-shaped data (JSON trees, collaborative text) fits your domain.

**Avoid when:**
- Your data is large, relational, or query-heavy — no indexes, no partial
  loads, whole-document materialization.
- You need maximum text-editing performance and small documents more than
  history — Yjs's tombstone GC trades history for size.
- A central authoritative server already exists and can arbitrate writes;
  CRDT complexity buys you little there.
- You need server-enforced invariants or permissions inside the merge — CRDTs
  merge everything; authorization must live elsewhere.

## Alternatives

- yjs/yjs — use when raw performance, small documents, and the largest
  editor-binding ecosystem matter more than retained history.
- loro-dev/loro — newer Rust CRDT with time-travel and rich types; evaluate
  for Automerge-like semantics with different performance tradeoffs.
- josephg/diamond-types — plain-text sequence CRDT focused on speed; use for
  text-only collaboration, not JSON documents.
- share/sharedb — operational transformation with a central server; use when
  an authoritative server exists and you want mature JSON OT instead of CRDTs.
- electric-sql/electric — Postgres-based sync engine; use when your data is
  relational and server-backed rather than document-shaped.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x (classic) | 2017 | Original pure-JavaScript implementation (now `automerge-classic`); proved the API, struggled on performance. |
| automerge-rs | 2019-12 | This repository created as the Rust port. |
| 2.0 | 2023-01 | Rust core + WASM becomes canonical; large speed/size gains over classic[^2]. |
| automerge-repo 1.0 | 2023 | Storage/network/repo layer GA — the recommended entry point[^5]. |
| 3.0 | 2025 | ~10× memory reduction; unified text handling[^3]. |

## References

[^1]: Kleppmann, Wiggins, van Hardenberg, McGranaghan — "Local-first software: You own your data, in spite of the cloud", Ink & Switch, 2019. https://www.inkandswitch.com/local-first/
[^2]: Automerge blog, "Automerge 2.0" — 2023-01. https://automerge.org/blog/automerge-2/
[^3]: Automerge blog, "Automerge 3". https://automerge.org/blog/automerge-3/
[^4]: Automerge README, "Status" — Rust API described as low-level and not well documented; `autosurgeon` recommended for Rust apps. https://github.com/automerge/automerge#status
[^5]: automerge-repo — batteries-included repo/sync/storage layer. https://github.com/automerge/automerge-repo
[^6]: Automerge binary format specification. https://automerge.org/automerge-binary-format-spec
[^7]: Litt, Lim, Kleppmann, van Hardenberg — "Peritext: A CRDT for Rich-Text Collaboration", Ink & Switch, 2021. https://www.inkandswitch.com/peritext/

## Tags

javascript, rust, wasm, crdt, local-first, offline-first, sync, distributed-systems, collaboration, data-structures
