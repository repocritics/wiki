# oklog/ulid

> A Go implementation of ULID — 128-bit, time-prefixed, lexicographically sortable identifiers that stay binary-compatible with UUIDs.

[GitHub repo](https://github.com/oklog/ulid) ·
[License: Apache-2.0](https://github.com/oklog/ulid/blob/main/LICENSE)

## Overview

ULID (Universally Unique Lexicographically Sortable Identifier) is a 128-bit ID format that splits its bits into a 48-bit millisecond timestamp followed by 80 bits of entropy, canonically rendered as a 26-character Crockford base32 string[^1]. The point is to keep the collision-resistance of a random UUID while making IDs sort in roughly creation order — so they behave well as primary keys and index keys instead of scattering B-tree inserts the way UUIDv4 does. `oklog/ulid` is the Go port of the original JavaScript reference implementation, with the binary layout added[^2].

The package is small, dependency-free, and has been stable for years — the value is in the format and the encoding correctness, not in a large API surface. It is widely used across the Go ecosystem (it is a common transitive dependency of observability and database tooling) and is maintained under the `oklog` organization, which also produced the OK Log distributed log project.

The defining tension is that ULID hands you the entropy source rather than choosing one for you. `ulid.New` takes an `io.Reader`, which lets you trade off cryptographic strength, throughput, and monotonicity — but it also means the "obvious" call is not the safe one, and newcomers routinely wire up a non-thread-safe `math/rand` source or forget monotonicity. The README says so plainly: this design "can be a bit confusing to newcomers"[^2].

## Getting Started

```shell
go get github.com/oklog/ulid/v2
```

```go
package main

import (
	"fmt"

	"github.com/oklog/ulid/v2"
)

func main() {
	// Simplest path: process-global, monotonic, pseudo-random entropy.
	id := ulid.Make()
	fmt.Println(id) // e.g. 01G65Z755AFWAKHE12NY0CQ9FH
}
```

For control over timestamp and entropy, use `ulid.New` with an explicit reader:

```go
entropy := ulid.Monotonic(crypto_rand.Reader, 0) // monotonic within a millisecond
ms := ulid.Timestamp(time.Now())
id, err := ulid.New(ms, entropy)
```

## Architecture / How It Works

A ULID is 16 octets: a 48-bit big-endian Unix-millisecond timestamp, then 80 bits of entropy. The string form is the same 128 bits re-encoded in Crockford base32 (alphabet `0123456789ABCDEFGHJKMNPQRSTVWXYZ`, which drops I, L, O, U to reduce transcription errors), producing 26 characters: 10 for the timestamp, 16 for the entropy[^1]. Because the byte order is big-endian with the timestamp first, lexical order of the encoded string matches chronological order — this is the whole trick.

The Go type `ulid.ULID` is a `[16]byte` array, so it is comparable, copyable, and cheap to pass by value. It implements the standard `encoding.TextMarshaler`, `BinaryMarshaler`, `sql.Scanner`, and `driver.Valuer` interfaces, so it drops into `database/sql` and JSON without adapters. The 48-bit timestamp is why ULIDs are UUID-compatible in storage: 128 bits fits exactly into a UUID column or `uuid` Postgres type.

Entropy is a strategy, not a constant. Three common configurations:

- **`ulid.Make`** — process-global, locked, monotonic, `math/rand`-seeded. Convenient, thread-safe, not cryptographically secure.
- **Per-goroutine reader** — no lock contention, highest throughput, but no cross-goroutine monotonicity and weaker randomness guarantees. Often pooled via `sync.Pool`.
- **`crypto/rand` + `ulid.Monotonic`** — cryptographically secure and monotonic within a millisecond, at the cost of a mutex and slower generation.

Monotonicity is only guaranteed within a single millisecond and only through a `MonotonicReader`; two ULIDs made in the same millisecond from a plain random reader are ordered by raw entropy, i.e. effectively unordered. The monotonic reader increments the previous entropy value instead of redrawing, and returns an error if it would overflow the 80-bit space within that millisecond.

## Production Notes

- **Pick the entropy source deliberately.** The default `math/rand`-based path is not suitable where IDs must be unpredictable (session tokens, object references exposed to users). Use `crypto/rand`. Conversely, `crypto/rand` under a shared mutex is the slow path — for high-ingest ID minting, per-goroutine or pooled entropy is standard.
- **Timestamps leak.** The first 48 bits are a readable creation time. That is a feature for sortability and a liability for anything you don't want to disclose the creation instant of. Do not treat a ULID as an opaque secret.
- **Monotonicity has limits.** It holds only within one millisecond and only via a monotonic reader; across process restarts, clock skew, or multiple hosts there is no global ordering — IDs from two machines interleave only as finely as their clocks agree. ULID is not a distributed sequence.
- **v1 vs v2 module paths.** Current code imports `github.com/oklog/ulid/v2`. Legacy code on the un-versioned `github.com/oklog/ulid` path is a different, older API and will not deduplicate against v2 in the module graph — mixed imports pull both. Standardize on `/v2`.
- **UUIDv7 overlaps the use case.** RFC 9562 (2024) standardized UUIDv7, a time-ordered UUID that solves the same B-tree-locality problem inside the UUID format[^3]. For new systems that want time-sortable IDs but prefer to stay in canonical 36-char UUID text and native `uuid` DB types, UUIDv7 is now the mainstream alternative and worth weighing before adopting ULID's base32 string form.
- **Benchmark numbers in the README are old.** The published figures were measured on a 2016-era Core i7 with Go 1.8beta[^2]; treat them as directional (generation is cheap, binary marshal is near-free), not as current absolute performance.

## When to Use / When Not

**Use when:**
- You want database primary keys that insert in roughly time order to keep index locality good.
- You need IDs that are sortable, URL-safe, case-insensitive, and shorter to read than a 36-char UUID.
- You want to stay 128-bit / UUID-compatible in storage while getting a nicer string form.

**Avoid when:**
- You need strong unpredictability and don't want to think about entropy — a plain `crypto/rand` UUIDv4 has fewer footguns.
- You require a globally ordered or gap-free sequence — ULID is only millisecond-monotonic and only within one process.
- You'd rather stay in canonical UUID text and DB types — UUIDv7 gives time-ordering there.
- You don't actually need time ordering — as the README says, if so "there's no reason to use ULIDs."[^2]

## Alternatives

- google/uuid — the standard Go UUID library; use it when you want plain UUIDv4 and don't need lexical time ordering.
- gofrs/uuid — fuller UUID variant support (including v6/v7); use it when you want time-ordered IDs but in canonical UUID form.
- segmentio/ksuid — 27-char, 160-bit, second-precision sortable IDs; use it when a 4-byte second timestamp and larger entropy space suit you better than ULID's millisecond field.
- rs/xid — 12-byte / 20-char MongoDB-ObjectID-style sortable IDs; use it when you want smaller IDs and are fine with a machine+pid+counter scheme instead of pure entropy.
- bwmarrin/snowflake — 64-bit coordinated Snowflake IDs; use it when you need compact int64 keys and can run coordinated worker IDs.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-12-06 | Repository created; Go port of ulid/javascript with binary format[^2]. |
| v1.x | — | Original un-versioned module path `github.com/oklog/ulid`. |
| v2.x | — | Module `github.com/oklog/ulid/v2`; reworked entropy/monotonic API and `ulid.Make` convenience helper. |
| latest | 2025-06-09 | Most recent push to `main`; project remains stable and low-churn[^4]. |

(Exact tag dates for v1/v2 are omitted here rather than guessed; see the GitHub releases page.)

## References

[^1]: ULID specification, ulid/javascript. https://github.com/ulid/javascript
[^2]: oklog/ulid README (background, usage, spec, benchmarks). https://github.com/oklog/ulid
[^3]: RFC 9562, "Universally Unique IDentifiers (UUIDs)", incl. UUIDv7 (2024). https://www.rfc-editor.org/rfc/rfc9562
[^4]: GitHub API repository metadata for oklog/ulid (stars, forks, license, last push), fetched 2026-07-15.

## Tags

go, ulid, uuid, identifier, sortable-id, distributed-systems, primary-key, crockford-base32, entropy, database-keys
