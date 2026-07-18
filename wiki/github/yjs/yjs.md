# yjs/yjs

> Shared data types for building collaborative software — the most widely deployed CRDT implementation in the JavaScript ecosystem.

[GitHub repo](https://github.com/yjs/yjs) ·
[Official docs](https://docs.yjs.dev) ·
[License: MIT](https://github.com/yjs/yjs/blob/main/LICENSE)

## Overview

Yjs is a CRDT (Conflict-free Replicated Data Type) framework by Kevin Jahns, started in 2014 as research at RWTH Aachen and formalized in the 2016 YATA paper[^1]. It exposes *shared types* — `Y.Map`, `Y.Array`, `Y.Text`, and XML types — that behave like ordinary containers but automatically merge concurrent edits from any number of peers without conflicts. The core is network-agnostic: transport (WebSocket, WebRTC, anything) and persistence are separate provider modules. At ~22k stars it is the de facto standard for real-time collaborative text editing on the web, used by JupyterLab, Evernote, GitBook, Linear, Proton Docs, AFFiNE, and the entire Tiptap/Hocuspocus ecosystem.

The defining tradeoff versus Automerge, its main rival: Yjs optimizes for document size and merge speed by garbage-collecting deleted content rather than keeping full edit history[^2]. Documents stay small and syncs stay fast — dmonad's own crdt-benchmarks show order-of-magnitude gaps on some workloads[^3] — but you give up git-like historical introspection.

Second thing to know: Yjs is substantially a one-person project, maintained by Jahns and funded by GitHub Sponsors and support contracts. Development is active (v14 release candidates shipping weekly as of mid-2026, last push within days), but the bus factor is real — partially mitigated by the independent Rust port (yrs) under the y-crdt org.

## Getting Started

```bash
npm install yjs y-websocket
```

```js
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'

const doc = new Y.Doc()
// connect this doc to a room; all docs in the room converge
const provider = new WebsocketProvider('wss://demos.yjs.dev/ws', 'my-room', doc)

const ytext = doc.getText('article')
ytext.observe(() => console.log(ytext.toString()))
ytext.insert(0, 'Hello')          // merges conflict-free with remote edits

const ymeta = doc.getMap('meta')
ymeta.set('title', 'Untitled')    // concurrent set on same key: last-writer-wins
```

Editor integration goes through a binding package (`y-prosemirror`, `y-quill`, `y-codemirror`, `y-monaco`; native in Lexical/BlockSuite), not against `Y.Text` directly.

## Architecture / How It Works

Every insertion becomes an `Item` struct with a globally unique ID `(clientID, clock)` — a random client identifier plus a per-client Lamport clock. Items form doubly-linked lists under their parent type; each item records the IDs of its left and right origin at insertion time, and the YATA integration algorithm[^1] uses those origins (with clientID as tiebreak) to place concurrent insertions in the same deterministic order on every peer.

Sync is delta-based: each peer summarizes what it has as a **state vector** (map of clientID → clock), and `encodeStateAsUpdate(doc, remoteStateVector)` produces exactly the missing operations[^4]. Updates are encoded with the lib0 variable-length binary format — compact enough that whole-document snapshots routinely ship as single updates. Two wire formats exist (v1 and the more compressed v2); they are not interchangeable without conversion.

Deletions are tombstones. Content of deleted items is garbage-collected (replaced by lightweight GC structs) but delete-set metadata is retained forever, so documents grow monotonically with edit count, just slowly. Snapshots and version history require constructing the doc with `gc: false`, which forfeits the size advantage.

Presence (cursors, user names) is deliberately *not* CRDT state: the Awareness protocol in `y-protocols` is ephemeral metadata that providers broadcast and expire — newcomers routinely miss this and try to store presence in shared types.

The ecosystem is a constellation of small repos: connection providers (y-websocket, y-webrtc, Hocuspocus, y-sweet, managed options like Liveblocks), persistence providers (y-indexeddb, y-redis, y-mongodb-provider), editor bindings, and ports — yrs (Rust) with wasm/Python/Ruby bindings shares the binary format, so a JS client can sync against a Rust server[^5].

## Production Notes

**The bundled y-websocket server is demo-grade** — in-memory, no auth, no horizontal scaling. Production deployments use Hocuspocus (auth hooks, webhooks, persistence), y-redis, y-sweet, or a managed service (Liveblocks, PartyKit, Velt). "Add a real backend" is the first real cost of adopting Yjs.

**Updates are opaque binary.** The server relays bytes it cannot cheaply inspect. Validation, schema enforcement, and fine-grained permissions (user A may edit field X but not Y) require decoding the document server-side, or accepting everything. Document-level auth is easy; sub-document authorization is a known hard problem in the ecosystem.

**Documents grow forever.** Tombstone metadata survives GC, so a long-lived, heavily edited doc slowly bloats. The only compaction is copying content into a fresh `Y.Doc`, which breaks CRDT continuity — old offline clients can no longer merge, and stored relative positions die. Plan a document lifecycle up front for anything with years of expected edits.

**`Y.Map` is last-writer-wins per key**, and values that are plain JSON objects are overwritten wholesale on concurrent edit — only nested *shared types* merge granularly. Deep plain objects inside a Y.Map are the most common data-loss-shaped surprise.

**No native move operation.** Concurrently moving a `Y.Array` element (delete + insert) can duplicate it. Ordered-tree and kanban-style apps need application-level schemes or helpers like y-utility's `YKeyValue`.

**Memory model.** The whole document lives in memory on every peer, including the server. Hosting thousands of active docs needs explicit load/unload eviction; naive implementations OOM. Subdocuments help partition large workspaces but provider support for them is inconsistent.

**v14 is imminent.** The stable 13.x line ran from 2020 to 13.6.31 (2026-05-28); v14.0.0 has been in release candidates through mid-2026[^6] — the first major bump in six years. Pin 13.6.x until your providers and bindings declare v14 support.

## When to Use / When Not

**Use when:**
- You are building collaborative text or structured-document editing — the ProseMirror/Tiptap, CodeMirror, Monaco, Quill, Lexical bindings are mature.
- You need offline-first sync with automatic conflict-free merge on reconnect (pair with y-indexeddb).
- You want transport flexibility: the same doc syncs over WebSocket, WebRTC p2p, or anything that moves bytes.
- Document size and sync latency matter more than complete edit history.

**Avoid when:**
- You need full auditable history, branching, or time travel by default — Automerge or Loro fit better.
- Your data is relational with server-side invariants (uniqueness, foreign keys) — CRDTs cannot enforce global invariants; use an authoritative server.
- You need per-field authorization enforced server-side.
- "Last save wins" or OT-with-central-server covers your actual concurrency needs — a CRDT adds complexity you won't use.

## Alternatives

- automerge/automerge — use instead when you need full change history and branching/merging semantics, and are willing to pay in document size.
- loro-dev/loro — newer Rust CRDT with rich-text, movable tree, and time travel; use when tree structures and history matter more than ecosystem maturity.
- share/sharedb — OT with an authoritative central server; use when you want server-side authority and JSON query subscriptions over p2p convergence.
- y-crdt/y-crdt — the Rust port (yrs), wire-compatible with Yjs; use for native backends or non-JS clients in a Yjs ecosystem.
- josephg/diamond-types — experimental, extremely fast plain-text CRDT; use for research or text-only workloads, not production apps.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014-07 | Repo created; early versions as RWTH Aachen research project. |
| — | 2016 | YATA paper published (ACM GROUP '16), formalizing the algorithm[^1]. |
| 13.0 | 2020-01 | Ground-up rewrite: modular providers, lib0 binary encoding. |
| 13.5 | 2021-02 | Subdocuments, v2 update encoding. |
| 13.6 | 2023-04 | Final stable line; maintained through 13.6.31 (2026-05-28). |
| 14.0 | 2026 (RC) | First major in six years; rc.24 published 2026-07-15[^6]. |

## References

[^1]: P. Nicolaescu, K. Jahns, M. Derntl, R. Klamma — "Near Real-Time Peer-to-Peer Shared Editing on Extensible Data Types", ACM GROUP 2016. https://dl.acm.org/doi/10.1145/2957276.2957310
[^2]: Kevin Jahns, "Are CRDTs suitable for shared editing?" — 2020. https://blog.kevinjahns.de/are-crdts-suitable-for-shared-editing/
[^3]: crdt-benchmarks — Yjs vs Automerge and others, maintained by the Yjs author. https://github.com/dmonad/crdt-benchmarks
[^4]: Yjs docs, "Document Updates" — state vectors and diff sync. https://docs.yjs.dev/api/document-updates
[^5]: y-crdt — Rust port (yrs) with wasm/Python/Ruby bindings. https://github.com/y-crdt/y-crdt
[^6]: Yjs releases. https://github.com/yjs/yjs/releases

## Tags

javascript, crdt, collaborative-editing, realtime, local-first, offline-first, p2p, sync, rich-text, data-structures
