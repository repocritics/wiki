# stretchr/objx

> Go package for reaching into `map[string]interface{}` — JSON, config, and other loosely-typed data — with dot-path access and forgiving type coercion.

[GitHub repo](https://github.com/stretchr/objx) ·
[API docs](https://pkg.go.dev/github.com/stretchr/objx) ·
[License: MIT](https://github.com/stretchr/objx/blob/master/LICENSE)

## Overview

Objx is a small Go library from the Stretchr team (the authors of `testify`) for working with untyped, map-shaped data — the kind you get from `json.Unmarshal` into an `interface{}`, from a config blob, or from a URL query. Its core type, `objx.Map`, is a plain `map[string]interface{}` with a `Get` method that walks nested structures using dot-and-bracket notation (`m.Get("places[0].latlng")`) and returns a `Value` wrapper. From that `Value` you either test the type (`IsStr`, `IsBool`, ...) or coerce it (`Str()`, `Int()`, `Float64()`), optionally supplying a default that is returned when the key is missing or the type does not match[^1].

The defining tradeoff is deliberate: objx trades Go's compile-time type safety for runtime convenience. Nothing about a path is checked by the compiler, missing keys never error (they yield a zero/default `Value`), and coercion is "optimistic" — the non-`Must` accessors never panic, they just fall back. This makes objx pleasant for exploratory access to unpredictable data and dangerous for code where a typo or shape change should be a hard failure. The `Must*` variants (`MustFromJSON`) exist for the cases where you do want a panic on bad input.

In practice most Go developers acquire objx transitively rather than choosing it: it is a dependency of `stretchr/testify`'s mock package and of `vektra/mockery`, so it shows up in a large share of Go `go.sum` files without ever being imported directly[^2]. As a direct dependency it occupies a narrow niche that has shrunk since generics (Go 1.18) and better struct-based JSON decoding became idiomatic. The ~865 stars understate its reach and overstate its role — it is infrastructure, maintained in a slow, stable maintenance cadence rather than actively grown.

## Getting Started

```bash
go get github.com/stretchr/objx
```

```go
package main

import (
	"fmt"

	"github.com/stretchr/objx"
)

func main() {
	// Build a Map from JSON. Non-Must variant returns (Map, error).
	m := objx.MustFromJSON(`{"name":"Mat","age":30,"places":[{"latlng":"1,2"}]}`)

	name := m.Get("name").Str()           // "Mat"
	age := m.Get("age").Int()             // 30
	loc := m.Get("places[0].latlng").Str()// "1,2"

	// Missing key with an explicit default:
	nick := m.Get("nickname").Str(name)   // falls back to "Mat"

	fmt.Println(name, age, loc, nick)
}
```

## Architecture / How It Works

Objx is intentionally shallow. There are three moving parts:

1. **`Map`** — a type alias over `map[string]interface{}` (the code refers to this as an "MSI"). Because it is the underlying map, you can `range` it, index it, and pass it anywhere a `map[string]interface{}` is expected, with no conversion. Constructors (`New`, `FromJSON`, `FromBase64`, `FromSignedBase64`, `FromURLQuery`) all produce one[^1].
2. **`Value`** — a struct wrapping a single `interface{}`. Every accessor (`Str`, `Int`, `Bool`, `Slice`, `ObjxMap`, ...) is a type assertion with a default fallback, and each has an `Is*` predicate and an `*Slice` companion. A large part of the codebase — `type_specific_codegen.go` — is generated from templates via `go generate` to produce the full accessor matrix for every supported primitive.
3. **Path access** — `Get` parses the string path (dot segments and `[index]` brackets) on every call and descends through nested maps and slices. Parsing is string/regex-based, not a compiled expression, so a `Get` is not free.

On top of `Value` there is a small collection layer for slices: `Each`, `Where`, `Group`, `Replace`, and `Collect` iterate `[]interface{}` values. Serialization helpers round-trip to JSON, base64, and URL-query form. The `FromSignedBase64` / `SignedBase64` pair attach an HMAC-style checksum computed with SHA-1 — this is a tamper-evidence convenience (e.g. for cookies), not a modern cryptographic guarantee, and should not be treated as one.

There is no schema, no validation layer, and no reflection into user structs. Everything is `interface{}` in and `interface{}` out; objx's whole job is making the assertions and nil checks less verbose at the call site.

## Production Notes

- **No compile-time safety, by design.** A renamed JSON field or a wrong path becomes a silently-returned default, not an error. In services that persist or forward that data, this can mask bugs indefinitely. Prefer decoding into a typed struct wherever the shape is known; reserve objx for genuinely dynamic data.
- **`Get` is not for hot paths.** Each call re-parses the path string and walks the structure with type assertions. For a config read at startup this is irrelevant; inside a per-request loop over large documents it allocates and adds up. Cache the resolved `Value` rather than re-`Get`-ing the same path.
- **Silent defaults hide missing data.** `m.Get("a.b.c").Int()` returns `0` whether `c` is absent, null, or actually zero. If you need to distinguish "missing" from "zero", check `IsNil()` / the `Is*` predicate before coercing — the terse form throws that information away.
- **Signed base64 uses SHA-1.** `FromSignedBase64` verifies an appended SHA-1 hash. It defends against accidental corruption and naive tampering, not against a determined attacker; do not use it as an authentication or integrity primitive for untrusted input.
- **Mostly a transitive dependency.** It arrives via `testify/mock` and `mockery`. Removing it from your direct dependencies rarely removes it from your module graph, so "we don't use objx" is often untrue at the `go.sum` level[^2].
- **Maintenance cadence, not active development.** Releases are infrequent (v0.5.3 in October 2025, v0.5.2 over a year before it) and changes trend toward Go-version support and housekeeping rather than new API. The `go.mod` targets `go 1.20`, and the project states it supports the three most recent Go majors[^3]. It is stable and low-risk, but do not expect it to evolve.

## When to Use / When Not

**Use when:**
- You are reading genuinely dynamic, schema-less data (arbitrary JSON, plugin config, webhook payloads) and want dot-path access without hand-writing nested type assertions.
- You already depend on it via testify/mockery and want to reuse it rather than add another helper.
- The convenience of optimistic defaults outweighs the loss of type errors — e.g. scripts, tooling, exploratory code.

**Avoid when:**
- The data shape is known: decode into a struct with `encoding/json` and get compile-time checking and better performance for free.
- You need missing-vs-zero distinctions or hard failure on malformed input without sprinkling `Must` and `Is*` everywhere.
- You are on a modern Go codebase where generics and typed decoders already cover your access patterns and adding an untyped map layer is a step backward.

## Alternatives

- tidwall/gjson — read-only JSON path queries straight off a `[]byte`, no `interface{}` map; use when you only need to extract values from JSON and want speed.
- tidwall/sjson — companion setter for JSON documents; use when you need to mutate JSON by path without full unmarshaling.
- spf13/cast — the coercion half of objx (`ToInt`, `ToString`, ...) without the map/path layer; use when you only need forgiving type conversion.
- Standard `encoding/json` into structs — use whenever the shape is known; it is the idiomatic, type-safe, faster default.
- buger/jsonparser — zero-allocation JSON scanning by key path; use for high-throughput extraction where allocations matter.

## History

| Version | Date | Notes |
|---------|------|-------|
| (repo created) | 2013-09-10 | Initial commit under the Stretchr org[^4]. |
| v0.1.0 / v0.1.1 | 2018-06 | First semantic-version tags[^4]. |
| v0.2.0 | 2019-04-09 | Continued API and codegen work[^4]. |
| v0.3.0 | 2020-07-15 | Maintenance release[^4]. |
| v0.4.0 | 2022-05-08 | Go module maintenance[^4]. |
| v0.5.0 | 2022-10-15 | Dropped older Go versions; module modernization[^4]. |
| v0.5.2 | 2024-02-29 | Housekeeping / dependency and Go-support updates[^4]. |
| v0.5.3 | 2025-10-07 | Latest release; maintenance[^4]. |

## References

[^1]: Objx README and package overview — `objx.Map`, `Get`, and the `Value` accessor pattern. https://github.com/stretchr/objx and https://pkg.go.dev/github.com/stretchr/objx
[^2]: objx is a dependency of `stretchr/testify`'s mock package and of `vektra/mockery`. https://github.com/stretchr/testify · https://github.com/vektra/mockery
[^3]: `go.mod` targets `go 1.20`; README states support for the three most recent major Go versions. https://github.com/stretchr/objx/blob/master/go.mod
[^4]: Release tags and dates from the GitHub Releases API for stretchr/objx (v0.1.0 through v0.5.3). https://github.com/stretchr/objx/releases

## Tags

go, golang, json, map, data-access, type-coercion, dot-notation, utility-library, testify-ecosystem, dynamic-data
