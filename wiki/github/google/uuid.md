# google/uuid

> Go package that generates and parses UUIDs per RFC 4122 / RFC 9562 and DCE 1.1, storing each value as a fixed 16-byte array.

[GitHub repo](https://github.com/google/uuid) ·
[Go Reference](https://pkg.go.dev/github.com/google/uuid) ·
[License: BSD-3-Clause](https://github.com/google/uuid/blob/master/LICENSE)

## Overview

`google/uuid` is the de facto standard UUID library for Go. It descends from
`github.com/pborman/uuid` (originally `code.google.com/p/go-uuid`), and its one
structural departure from that ancestor defines the whole API: a `UUID` is a
`[16]byte` array, not a byte slice[^1]. That makes the type comparable,
usable as a map key, and copyable without aliasing, at the cost of losing the
ability to represent an "invalid" UUID — the zero value is the Nil UUID, and
parse failures are surfaced as errors rather than as a distinct invalid state.

The package covers the full spec surface: version 1 (time + node), version 2
(DCE Security), versions 3 and 5 (namespaced MD5 / SHA-1), version 4 (random),
and the newer time-ordered versions 6 and 7 plus the custom version 8[^2]. It
has no external dependencies and pulls only from the standard library, which is
a large part of why it appears in so many dependency trees — it is effectively
free to add.

The defining tension is between ergonomics and failure handling. The headline
constructor `uuid.New()` returns a bare `UUID` with no error, so it *panics* if
the system entropy source fails. That is convenient in the 99% case and a
latent footgun in constrained or sandboxed environments where `crypto/rand` can
block or error.

## Getting Started

```sh
go get github.com/google/uuid
```

```go
package main

import (
	"fmt"

	"github.com/google/uuid"
)

func main() {
	id := uuid.New()                 // random v4; panics if entropy fails
	fmt.Println(id.String())         // 550e8400-e29b-41d4-a716-446655440000

	id7, err := uuid.NewV7()         // time-ordered v7 (sortable)
	if err != nil {
		panic(err)
	}
	fmt.Println(id7)

	parsed, err := uuid.Parse("550e8400-e29b-41d4-a716-446655440000")
	if err != nil {
		panic(err)
	}
	fmt.Println(parsed.Version(), parsed.Variant())
}
```

Use `uuid.NewString()` when you only need the string form, and
`uuid.MustParse()` only for compile-time-known constants — it panics on bad
input.

## Architecture / How It Works

The core type is `type UUID [16]byte`. Because it is an array, equality
(`==`), assignment, and map-key use work without any helper methods, and
`Bytes()` / `String()` are cheap formatters over the fixed layout. `Parse`
accepts the canonical hyphenated form, the URN prefix
(`urn:uuid:...`), the brace-wrapped Microsoft form, and the raw 32-hex form;
`ParseBytes` does the same over a `[]byte` to avoid a string allocation.

Randomness is centralized behind a package-level reader. `NewRandom` (and thus
`New`) reads from `crypto/rand.Reader` by default. `SetRand(io.Reader)` swaps
that source globally — useful for deterministic tests, but it is not safe to
call concurrently with UUID generation. `EnableRandPool()` batches entropy
into a 16-value buffer to cut the number of `read` syscalls under high v4
throughput[^3]; it trades a small amount of memory and a mutex for fewer round
trips to the kernel.

Time-based versions carry more machinery. Version 1 combines a 60-bit
timestamp, a clock sequence, and a 48-bit node ID that defaults to the host MAC
address, guarded by a package mutex so the clock sequence advances
monotonically. Versions 6 and 7 reorder the timestamp bytes so that the
resulting UUIDs sort lexicographically in creation order — the property that
makes them attractive as database primary keys. Namespaced versions 3 and 5
are pure hash functions over `(namespace, name)` and are deterministic, with
the well-known DNS/URL/OID/X500 namespace constants provided.

## Production Notes

**`uuid.New()` panics on entropy failure.** It is defined as
`Must(NewRandom())`. In almost all deployments `crypto/rand` never fails, but
inside minimal containers, early-boot init, seccomp-filtered sandboxes, or some
WASM targets it can. If any of those apply, call `NewRandom()` and handle the
error instead of the panicking wrapper.

**Prefer v7 over v4 for database keys.** Random v4 UUIDs scatter inserts across
a B-tree index and fragment it; v7's time-ordered prefix keeps inserts
append-mostly, which materially reduces page splits and index bloat on large
tables. This is the single most common reason teams migrate constructors. Note
that v7's leading timestamp *leaks approximate creation time*, which is
occasionally an information-exposure concern.

**Version 1 leaks host identity.** The default node ID is the machine MAC
address, so v1 UUIDs can expose which host minted them and roughly when. Use
`SetNodeID` with a random node, or avoid v1 entirely in favor of v4/v7, if that
matters.

**`SetRand` / `SetNodeID` / `SetClockSequence` are process-global and not
concurrency-safe with generation.** Configure them once at startup before any
goroutine starts issuing UUIDs. Re-tuning them at runtime is a data race.

**Comparisons are value comparisons.** Because `UUID` is an array, `a == b` and
using a UUID as a map key just work — but a `[]UUID` copied around is copied by
value, which is cheap here (16 bytes) yet worth knowing versus slice-backed
libraries.

**API stability.** The package has been effectively frozen in shape for years;
`v1.x` releases add versions and helpers without breaking the type. Upgrades
across the `v1` line have been low-risk in practice.

## When to Use / When Not

**Use when:**
- You need standards-compliant UUIDs in Go with zero extra dependencies.
- You want a comparable, map-key-safe value type rather than a byte slice.
- You need time-ordered v7 identifiers for database primary keys.
- You want deterministic namespaced IDs (v3/v5) for content addressing.

**Avoid / reconsider when:**
- You run in an environment where `crypto/rand` may fail and you use the
  panicking `New()` without guarding it.
- You need k-sortable IDs shorter than 128 bits — ULID or a Snowflake-style
  scheme is more compact.
- You specifically need `pborman/uuid`'s slice-based `UUID` and its notion of
  an invalid value.

## Alternatives

- pborman/uuid — the slice-based ancestor; use it when you need an explicit
  invalid-UUID state or must match its older API.
- gofrs/uuid — community fork of the original satori library with a strict,
  error-returning API; use it when you want no panicking constructors at all.
- oklog/ulid — 128-bit, lexicographically sortable, Crockford-base32 encoded;
  use it when you want a compact sortable ID and don't need RFC UUID format.
- segmentio/ksuid — 27-char, time-prefixed sortable IDs; use it when a
  URL-friendly k-sortable string matters more than UUID compatibility.
- rs/xid — 12-byte, MongoDB-ObjectID-style globally unique IDs; use it when you
  want smaller-than-UUID identifiers with an embedded timestamp.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2018-08-28 | First tagged release under `github.com/google/uuid`[^1]. |
| v1.1.0 | 2018-11-20 | Additional constructors and parsing helpers. |
| v1.2.0 | 2021-01-22 | Maintenance and API additions. |
| v1.3.0 | 2021-07-12 | `EnableRandPool()` for batched v4 entropy reads[^3]. |
| v1.4.0 | 2023-10-26 | Maintenance release. |
| v1.5.0 | 2023-12-12 | Time-ordered version 6 and version 7 support[^2]. |
| v1.6.0 | 2024-01-23 | Tracks the RFC 9562 update to the UUID spec[^4]. |

## References

[^1]: google/uuid README — package origin in `pborman/uuid` and the 16-byte
array design. https://github.com/google/uuid/blob/master/README.md
[^2]: google/uuid package reference — supported versions (1–8) including v6/v7.
https://pkg.go.dev/github.com/google/uuid
[^3]: `EnableRandPool` documentation — batched randomness for v4 generation.
https://pkg.go.dev/github.com/google/uuid#EnableRandPool
[^4]: RFC 9562, "Universally Unique IDentifiers (UUIDs)" — obsoletes RFC 4122.
https://datatracker.ietf.org/doc/html/rfc9562

## Tags

go, uuid, rfc-4122, rfc-9562, identifiers, uuidv7, guid, standard-library, library, unique-id
