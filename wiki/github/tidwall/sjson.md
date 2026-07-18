# tidwall/sjson

> Set, update, and delete JSON values in Go by dot-path — without unmarshalling into a struct or map.

[GitHub repo](https://github.com/tidwall/sjson) ·
[License: MIT](https://github.com/tidwall/sjson/blob/master/LICENSE)

## Overview

SJSON is a small Go package for writing values into a JSON document addressed by
a dot-notation path, e.g. `sjson.Set(doc, "name.last", "Anderson")`. It is the
write-side companion to [gjson](https://github.com/tidwall/gjson) (reads) and
[jj](https://github.com/tidwall/jj) (a command-line tool); all three are by Josh
Baker (tidwall) and share the same path grammar[^1]. The package has stayed
deliberately tiny since its 2016 release — a couple of set/delete entry points,
an options struct, and byte-slice variants — and the public API has changed
little in that time.

The defining idea is that SJSON does not parse the whole document. To set one
field it scans the JSON text to locate the target position, then splices the new
value into the string, leaving everything else byte-for-byte untouched. This is
what makes it fast for point edits on large documents, and it is also the source
of every caveat: the library trusts that its input is already well-formed. It
does not validate the surrounding JSON, so a single edit to a malformed document
can produce malformed output without an error.

SJSON is for the case where you have JSON as text or bytes, want to change a
handful of fields, and do not want to define structs or round-trip through
`encoding/json`. It is not a general JSON object model, an ORM, or a schema
tool. If you need to build or transform whole documents, marshalling from Go
types is usually the better fit.

## Getting Started

```sh
go get -u github.com/tidwall/sjson
```

```go
package main

import (
	"fmt"

	"github.com/tidwall/sjson"
)

func main() {
	const doc = `{"name":{"first":"Janet","last":"Prichard"},"age":47}`

	// Update an existing value.
	out, _ := sjson.Set(doc, "name.last", "Anderson")

	// Append to an array with the -1 key.
	out, _ = sjson.Set(out, "friends.-1", "Sara")

	// Delete a value.
	out, _ = sjson.Delete(out, "age")

	fmt.Println(out)
	// {"name":{"first":"Janet","last":"Anderson"},"friends":["Sara"]}
}
```

`Set` returns a new string and an error; it never mutates its input. Paths are
dot-separated keys (`name.last`), array indices (`children.1`), `-1` to append,
and a leading colon (`users.:2313`) to force a numeric string as an object key
rather than an array index. `.` and `:` inside a key are escaped with backslash.

## Architecture / How It Works

The core operation is a splice, not a parse-modify-serialize cycle. `Set` walks
the raw JSON looking for the path segments in order. When it finds the insertion
point it computes the byte range to replace and concatenates
`prefix + newValue + suffix`. Missing intermediate objects and arrays are created
on the way (setting `a.b.c` into `{}` produces `{"a":{"b":{"c":...}}}`), and
appending past the end of an array back-fills with `null`.

Values are encoded by type: strings, numbers, bools, and `nil` are written
directly; anything the package does not recognize is handed to
`encoding/json`'s marshaller. There is also a `SetRaw` family that inserts a
pre-serialized JSON fragment verbatim, which is how you avoid double-encoding
when you already hold valid JSON text.

Two options change the allocation behavior via `sjson.SetOptions` /
`sjson.SetBytesOptions`:

- **`Optimistic`** — assume the result is roughly the same size as the input and
  pre-size the buffer accordingly, avoiding a growth reallocation in the common
  in-place-update case.
- **`ReplaceInPlace`** — allow the output to reuse the input's backing array.
  With the byte-slice APIs this can make a same-size replacement zero-allocation.
  It is destructive: the input buffer may be overwritten, so it must not be used
  when the original bytes are still needed elsewhere.

Because the algorithm is a textual scan, its cost scales with the distance from
the start of the document to the edit point, not with total document size for
edits near the front. It shares gjson's path grammar but only the parts that
identify a single location — SJSON has no wildcards, queries, or modifiers,
since "set every element matching X" has no unambiguous meaning.

## Production Notes

- **Input must be valid JSON.** SJSON does not validate. The README states plainly
  that invalid JSON "will not panic, but it may return back unexpected results."
  If your input can be untrusted or malformed, validate first (e.g.
  `gjson.Valid` or `json.Valid`) before setting.
- **`ReplaceInPlace` aliases memory.** It reuses the input buffer, so the source
  slice can be mutated out from under you. Only use it when you own the bytes and
  will not read the pre-edit version again. Getting this wrong produces
  data-dependent corruption that is hard to reproduce.
- **Key ordering and formatting are not normalized.** SJSON preserves the
  surrounding text, including whitespace and existing key order, and inserts new
  keys at a deterministic position. It is not a canonicalizer; do not rely on it
  to produce stable, sorted output for hashing or diffing.
- **Repeated edits re-scan each time.** `Set` is one path per call. Applying many
  edits to a large document means many scans and many intermediate strings; the
  byte APIs with `ReplaceInPlace` mitigate the allocation cost but not the
  re-scan. For bulk construction, marshalling from a Go value is often cheaper.
- **Numeric object keys are a footgun.** `users.2313` treats `2313` as an array
  index. To address an object key that looks like a number you must use the colon
  form `users.:2313`. This surprises people migrating from map-based access.
- **The published benchmarks are old.** The README's numbers were run on Go 1.7;
  treat them as directional (SJSON far ahead of map/struct round-trips, zero-alloc
  with `ReplaceInPlace`) rather than exact for current Go runtimes[^2].

## When to Use / When Not

**Use when:**
- You hold JSON as text/bytes and need to change a few fields by path.
- You want to avoid defining structs or unmarshalling into `map[string]interface{}`.
- You are already using gjson for reads and want a matching write path.
- You need low-allocation point edits on large documents (with the byte + in-place APIs).

**Avoid when:**
- Your input may be malformed or untrusted and you cannot validate it first.
- You are constructing or transforming whole documents — marshal from Go types instead.
- You need canonical/sorted output, JSON Patch/Merge-Patch semantics, or schema validation.
- You need to set many values matching a query; SJSON addresses one concrete location per call.

## Alternatives

- tidwall/gjson — the read-side companion; pair with sjson rather than replace it.
- buger/jsonparser — read and `Set`/`Delete` on `[]byte` with a similar no-unmarshal philosophy; lower-level, more manual.
- valyala/fastjson — parses into a mutable value tree you can edit then serialize; use when you make many edits per document.
- encoding/json (stdlib) — marshal from structs; use when you own the schema and build whole documents.
- evanphx/json-patch — RFC 6902 / 7386 patch application; use when edits arrive as standard JSON Patch operations.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-10-19 | First commit, released alongside gjson[^1]. |
| — | 2016–2020 | `Delete`, byte-slice APIs (`SetBytes`, `SetRawBytes`), and `SetOptions` (`Optimistic`, `ReplaceInPlace`) added. |
| current | 2026-05-19 | Latest commit on `master`; API stable, MIT-licensed, ~2.7k stars. |

Exact tag dates between the initial release and today are omitted here rather
than guessed; the package's surface has been stable enough that most code
written against early versions still compiles.

## References

[^1]: SJSON README and path syntax, tidwall/sjson. https://github.com/tidwall/sjson
[^2]: Performance section (benchmarks run on Go 1.7), tidwall/sjson README. https://github.com/tidwall/sjson#performance

## Tags

go, json, json-manipulation, serialization, dot-notation, zero-allocation, performance, tidwall, library, no-unmarshal
