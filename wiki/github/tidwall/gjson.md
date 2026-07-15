# tidwall/gjson

> Read-only JSON value extraction for Go via dot-notation paths — no structs, no full unmarshal, near-zero allocation for a single lookup.

[GitHub repo](https://github.com/tidwall/gjson) ·
[Playground](https://tidwall.com/gjson-play) ·
[License: MIT](https://github.com/tidwall/gjson/blob/master/LICENSE)

## Overview

GJSON is a Go package by Josh Baker (tidwall) that pulls values out of a JSON
document by scanning its bytes directly, without unmarshalling into a struct or
`map[string]interface{}`. You hand it a document and a dot-notation path
(`gjson.Get(json, "name.last")`) and it walks the raw bytes until the value is
found, then returns immediately[^1]. For a single field lookup this is one line
of code and, for scalar paths, zero heap allocations.

The library is read-only by design. Its companion package
[sjson](https://github.com/tidwall/sjson) handles set/delete, and the
[jj](https://github.com/tidwall/jj) command-line tool wraps both. GJSON is the
extraction half of a deliberately split toolkit, and it is the read layer under
tidwall's own datastores (BuntDB, Tile38). It sits alongside `encoding/json`
rather than replacing it: you reach for GJSON when you need a handful of fields
out of a large or unpredictable payload, and for `encoding/json` (or a faster
drop-in) when you need full round-trip (un)marshalling into typed values.

The defining tradeoff is scan-per-query versus parse-once. GJSON does no schema
binding and, by default, no validation — malformed input does not panic but can
return silently wrong results[^2]. That makes it fast and forgiving, but it
pushes correctness and (for untrusted input) safety onto the caller.

## Getting Started

```sh
go get -u github.com/tidwall/gjson
```

```go
package main

import (
	"fmt"

	"github.com/tidwall/gjson"
)

const doc = `{"name":{"first":"Janet","last":"Prichard"},"age":47}`

func main() {
	last := gjson.Get(doc, "name.last")
	fmt.Println(last.String()) // Prichard

	// Array query: first friend whose last name is "Murphy"
	// friends.#(last=="Murphy").first
	if v := gjson.Get(doc, "name.first"); v.Exists() {
		fmt.Println(v.String()) // Janet
	}
}
```

For `[]byte` input use `gjson.GetBytes(data, path)` — it avoids the
`string(data)` copy that `Get` would force[^1].

## Architecture / How It Works

GJSON never builds a full parse tree. `Get` runs a single left-to-right scan of
the document, matching one path component at a time and short-circuiting the
moment the target is located; bytes after the match are never examined. The
return value is a `Result` struct carrying `Type`, `Str`, `Num`, `Raw` (the raw
JSON slice), and `Index` (byte offset of the raw value in the source, `0` when
unknown)[^1]. Objects and arrays come back as their raw JSON in `Raw`, which you
can re-query with `result.Get(...)` or expand via `result.Array()` / `.Map()`.

The path language is the real surface area:

- **Dot paths** with `*` / `?` wildcards on keys, and array indexing by number.
- **`#`** for array length (`children.#`) and for projecting a field across an
  array (`friends.#.first`).
- **Queries** `#(...)` (first match) and `#(...)#` (all matches) with `==`,
  `!=`, `<`, `<=`, `>`, `>=`, plus `%` (like) / `!%` (not like) pattern
  operators. The query bracket changed from `#[...]` to `#(...)` in v1.3.0 to
  disambiguate from multipaths; the old form still parses for backward
  compatibility[^1].
- **Modifiers** (v1.2+) — `@reverse`, `@pretty`, `@ugly`, `@flatten`, `@keys`,
  `@values`, `@group`, `@dig`, and others — composed with the pipe `|` for path
  chaining, and extensible via `gjson.AddModifier`[^1].
- **Multipaths** `{...}` / `[...]` to construct new objects/arrays inline, and
  the `..` prefix for JSON Lines documents.

Numbers are parsed into `Result.Num` as `float64`. `Result.Int()` and
`.Uint()` re-parse from `Raw` so they can read the full 64-bit integer range
without the float64 mantissa loss you would otherwise hit above 2^53[^1].
`GetBytes` exposes `Result.Index` so callers can reconstruct a zero-copy
sub-slice of the original buffer instead of allocating from `Raw`.

## Production Notes

**Validate untrusted input explicitly.** The `Get*`/`Parse*` functions assume
well-formed JSON. Bad JSON does not panic but "may return unexpected
results" — call `gjson.Valid(json)` (or the `@valid` modifier) before trusting
output from an external source[^2]. This is the single most common footgun.

**Security: keep to a recent release for adversarial input.** GJSON had a run of
denial-of-service advisories in the 2020–2021 window — panics / excessive CPU on
crafted paths and pathologically nested documents — addressed across the 1.6–1.9
releases[^3]. If you parse attacker-controlled JSON or attacker-controlled
paths, pin a current version and run `govulncheck`.

**Scan cost is per-`Get`.** Each top-level `Get`/`GetBytes` rescans from the
start of the document. Pulling twenty fields with twenty `Get` calls means
twenty scans. For multiple reads of one document, call `gjson.Parse(json)` once
and reuse `result.Get(...)`, or use `GetMany`/`GetManyBytes` to fetch several
paths in fewer passes. For a single field, GJSON beats a full unmarshal
handily; for whole-document access it does not.

**No streaming.** The entire document must be in memory as a string or byte
slice. There is no incremental/reader-based API, so multi-gigabyte inputs are
out of scope.

**Number precision.** `Result.Float()`/`.Num` is `float64`; for large integer
IDs use `.Int()`/`.Uint()` which read from `Raw`. Mixing them silently loses
precision on big values.

**Zero-copy `GetBytes` requires care.** The `Result.Index` sub-slice trick
aliases the source buffer — if you mutate or pool that buffer, the aliased
`Raw` bytes change underneath you.

## When to Use / When Not

**Use when:**
- You need a few fields out of a large or schema-unstable JSON payload.
- You want dot-path/query access (config lookups, webhook payloads, log lines).
- Allocation pressure matters and reads dominate over full decoding.
- You are already in the tidwall stack (sjson/jj/BuntDB/Tile38).

**Avoid when:**
- You need to (un)marshal whole documents into typed structs — use
  `encoding/json` or a faster drop-in.
- You need strict validation, JSON Schema, or canonical decoding guarantees.
- You must mutate JSON — that is sjson's job, not gjson's.
- Input is streamed and too large to hold in memory.

## Alternatives

- buger/jsonparser — same read-only, zero-alloc niche via a callback API; faster in some benchmarks, no query/modifier path language.
- tidwall/sjson — the companion write side; use it when you need set/delete rather than read.
- valyala/fastjson — mutable parse tree with a fast parser; use it when you also modify values and want an object model.
- bytedance/sonic — JIT-accelerated full marshal/unmarshal (amd64); use it as a drop-in `encoding/json` replacement when you need round-trips, not path extraction.
- json-iterator/go — near drop-in `encoding/json` replacement; use it when the goal is faster struct (un)marshalling.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-08 | First public release; dot-path `Get` over raw JSON[^4]. |
| v1.2 | — | Modifier functions (`@reverse`, `@pretty`, …) and pipe path chaining[^1]. |
| v1.3.0 | — | Query brackets moved `#[...]` → `#(...)`; multipath syntax[^1]. |
| 1.6–1.9 | 2020–2021 | Security hardening for DoS-class advisories on untrusted input[^3]. |
| ongoing | 2026-05 | Actively maintained; ~15.5k stars, mirror at Codeberg[^4]. |

## References

[^1]: GJSON README and path documentation. https://github.com/tidwall/gjson/blob/master/README.md and https://github.com/tidwall/gjson/blob/master/SYNTAX.md
[^2]: "The `Get*` and `Parse*` functions expects that the json is well-formed. Bad json will not panic, but it may return back unexpected results." — GJSON README, Validate JSON section. https://github.com/tidwall/gjson/blob/master/README.md
[^3]: GJSON security advisories (denial-of-service reports fixed across the 1.6–1.9 line). https://github.com/tidwall/gjson/security/advisories and https://pkg.go.dev/github.com/tidwall/gjson?tab=versions
[^4]: Repository metadata (created 2016-08-11, last push 2026-05-14, MIT, ~15,543 stars / 902 forks). https://github.com/tidwall/gjson

## Tags

go, golang, json, json-parser, json-path, query, read-only, zero-allocation, data-extraction, serialization, tidwall
