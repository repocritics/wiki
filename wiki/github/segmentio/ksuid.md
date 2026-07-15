# segmentio/ksuid

> K-sortable, coordination-free 20-byte unique IDs with a second-resolution timestamp prefix and 128 bits of entropy — Segment's Go reference implementation.

[GitHub repo](https://github.com/segmentio/ksuid) ·
[Blog: A brief history of the UUID](https://segment.com/blog/a-brief-history-of-the-uuid/) ·
[License: MIT](https://github.com/segmentio/ksuid/blob/master/LICENSE.md)

## Overview

KSUID ("K-Sortable Unique IDentifier") is an identifier format and Go library from Segment, first published in 2017[^1]. A KSUID is a 20-byte value: a 32-bit big-endian UTC timestamp (seconds since a custom 2014 epoch) followed by 128 bits of cryptographically-random payload. Because the timestamp is the high-order prefix and everything is big-endian, both the binary form and the 27-character base62 text form sort lexicographically by generation time. Running a column of KSUIDs through UNIX `sort` yields them in roughly creation order — no type-aware comparator required.

The design occupies a deliberate middle ground. UUIDv4 gives you coordination-free randomness but no ordering, which fragments database B-tree indexes on insert. Snowflake-style 64-bit IDs give you tight time ordering but require coordinated worker IDs and a central epoch authority. KSUID trades away 64-bit compactness for a self-contained, dependency-free ID that is both sortable and collision-safe without any coordination between generators. The 128-bit random component is larger than UUIDv4's 122 bits, so collision probability across independent generators is negligible.

The defining tension is time resolution. KSUID timestamps are **second-granular**. IDs generated within the same wall-clock second have no defined order relative to each other (their random payloads decide), so KSUID gives you coarse, best-effort ordering, not the millisecond or intra-process monotonicity that ULID and UUIDv7 offer. For log-scale "roughly when" sorting this is fine; for ordering a burst of events inside one second it is not, and that limitation drives most of the "should I use KSUID or X" decisions below.

## Getting Started

```bash
go get -u github.com/segmentio/ksuid
```

```go
package main

import (
	"fmt"

	"github.com/segmentio/ksuid"
)

func main() {
	id := ksuid.New()
	fmt.Println(id.String())      // e.g. 0ujtsYcgvSTl8PAuAdqWYSMnLOv (27 chars)
	fmt.Println(id.Time())        // generation time (second resolution)
	fmt.Printf("%x\n", id.Payload()) // 16 random bytes

	// Parse back from the text form.
	parsed, err := ksuid.Parse("0ujtsYcgvSTl8PAuAdqWYSMnLOv")
	if err != nil {
		panic(err)
	}
	fmt.Println(parsed.Time())
}
```

An optional CLI is included for generating and inspecting IDs:

```bash
go install github.com/segmentio/ksuid/cmd/ksuid@latest
ksuid -f inspect   # prints Time / Timestamp / Payload breakdown
```

## Architecture / How It Works

A KSUID is a fixed 20-byte array, not a variable-width type. That choice is load-bearing: the `KSUID` type is comparable, copyable, usable as a map key, and passable by value without heap allocation. Layout:

- **Bytes 0–3** — uint32 timestamp, big-endian, seconds since the KSUID epoch of **2014-05-13 UTC**. The custom epoch (rather than the Unix epoch) buys headroom: a uint32 of seconds from 2014 lasts until roughly the year 2150 before wrapping.
- **Bytes 4–19** — 128-bit payload from `crypto/rand` by default.

Big-endian ordering of the timestamp is what makes byte-wise comparison equal time comparison. The text representation is a full 27-character **base62** (`0-9A-Za-z`) encoding of the 20 bytes, chosen so IDs paste cleanly into URLs, filenames, and tokenizers without the delimiter-truncation problems hyphenated UUIDs cause. Base62 preserves lexical sort order, so text and binary sort identically.

Generation concurrency has two tiers. Package-level functions (`ksuid.New`, `ksuid.NewRandom`) are protected by a global mutex around the shared RNG, so they are safe to call from many goroutines but serialize on that lock. For hot single-goroutine loops, the `Sequence` type generates a monotonic run of KSUIDs sharing one timestamp with an incrementing payload, sidestepping both the mutex and same-second ambiguity within that sequence. RNG cost can be relaxed with `FastRander`, which seeds the standard-library PRNG from `crypto/rand` — faster, but the README explicitly warns it must not be used where ID unpredictability is a security property.

The `KSUID` type implements the common interfaces you need to drop it into existing stacks: `fmt.Stringer`, `sql.Scanner` / `driver.Valuer`, `encoding.BinaryMarshaler`/`Unmarshaler`, and `encoding.TextMarshaler`/`Unmarshaler` (so JSON round-trips as the 27-char string). Allocation-sensitive callers can use `Append` to parse into an existing value.

## Production Notes

- **Second resolution is the headline caveat.** Do not rely on KSUID ordering to sequence events that occur within the same second — use `Sequence`, a monotonic ID scheme, or a separate sequence column. Cross-machine ordering also inherits wall-clock skew: two hosts with drifting clocks will interleave incorrectly regardless of resolution.
- **Base62 is not free.** Encoding/decoding 20 bytes in base62 requires big-integer division across the whole value, which is measurably costlier than the Crockford base32 that ULID uses or plain hex. For most workloads this is noise; in extreme string-heavy hot paths it is a real difference, and generating the binary form (`New`) is far cheaper than repeatedly stringifying.
- **Storage footprint.** Binary KSUIDs are 20 bytes versus 16 for a UUID, and the 27-char text form versus a UUID's 36. If you store the text form in a database, index and row bloat come from that string, not the binary; prefer a 20-byte `BINARY(20)`/`bytea` column when ordering-in-index is the goal.
- **Index locality is the main reason to adopt it.** The time-prefix means new rows insert near the right edge of a B-tree instead of scattering like UUIDv4, reducing page splits and write amplification on high-insert tables. This is the concrete database win over random UUIDs.
- **`FastRander` weakens the security posture.** It is opt-in for a reason: predictable payloads are fine for uniqueness but not for "IDs must be unguessable." Keep the default CSPRNG for anything exposed to users.
- **Maintenance is quiet, not abandoned.** Segment was acquired by Twilio in 2020; the library is small, feature-complete, and battle-tested (Segment cites trillions of IDs in production), so commit cadence is intentionally low. Treat it as stable infrastructure, but do not expect rapid response to issues — the open-issue count reflects a mature project in maintenance mode rather than active feature work.

## When to Use / When Not

**Use when:**
- You want coordination-free IDs that also cluster by time in a database index, without running a Snowflake-style worker-ID scheme.
- You need IDs that are safe to paste into URLs, filenames, and logs (no delimiters, alphanumeric only).
- You want unguessable IDs (128 bits of CSPRNG entropy) that still sort roughly by creation time.
- You are already in Go and want a tiny, dependency-free, well-exercised implementation.

**Avoid when:**
- You need ordering finer than one second, or strict monotonicity within a burst — reach for ULID or UUIDv7.
- You want a standardized format your whole polyglot stack already supports — UUIDv7 is now an RFC and ships in most standard libraries.
- You need 64-bit IDs to fit a `bigint` column or wire budget — KSUID is 160 bits; use Snowflake/Sonyflake.
- You want the smallest possible IDs and don't need cryptographic entropy — `rs/xid` is 12 bytes.

## Alternatives

- oklog/ulid — millisecond-resolution, Crockford base32, optional monotonic generator. Use instead when you need finer time ordering or intra-millisecond monotonicity.
- google/uuid — standard UUID including v7 (time-ordered, RFC 9562). Use instead when you want a standardized, universally-supported format rather than a Segment-specific one.
- rs/xid — 12-byte MongoDB-ObjectID-style sortable ID. Use instead when compactness matters more than cryptographic entropy.
- bwmarrin/snowflake — 64-bit coordinated time-ordered IDs. Use instead when you must fit a `bigint` and can assign worker IDs.
- gofrs/uuid — full UUID toolkit (v1–v7). Use instead when you want UUIDv7's ordering with mature Go ergonomics and no custom epoch.

## History

KSUID does not use formal semantic-version release tags; it ships as a single feature-complete package. Milestones below are anchored to dates verifiable from the repository and design.[^3]

| Milestone | Date | Notes |
|-----------|------|-------|
| KSUID epoch | 2014-05-13 | Custom epoch baked into the format; gives ~136 years of uint32-second headroom. |
| Initial public release | 2017-05-11 | Repository published as the Go reference implementation.[^1] |
| Design write-up | 2017 | Segment's "A brief history of the UUID" popularized the format.[^2] |
| `OrNil` helpers | later | `ParseOrNil` / `FromBytesOrNil` / `FromPartsOrNil` added for ergonomic struct construction.[^3] |
| Maintenance mode | 2020– | Post-Twilio-acquisition; stable, low-cadence, still receiving occasional fixes (last push 2026-06). |

## References

[^1]: segmentio/ksuid repository, created 2017-05-11; README describes the library as the KSUID reference implementation. https://github.com/segmentio/ksuid
[^2]: Segment, "A brief history of the UUID." https://segment.com/blog/a-brief-history-of-the-uuid/
[^3]: ksuid README — format layout (20 bytes, 2014 epoch, base62 27-char text), `Sequence`/`FastRander`/`OrNil` APIs, and implemented standard-library interfaces. https://github.com/segmentio/ksuid/blob/master/README.md

## Tags

go, golang, unique-id, uuid, ksuid, k-sortable, distributed-systems, identifier, base62, database-keys
