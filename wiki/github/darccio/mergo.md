# darccio/mergo

> A Go library that fills the zero-value fields of a struct or map from another of the same type — the ubiquitous "apply defaults" helper for Go config code.

[GitHub repo](https://github.com/darccio/mergo) ·
[pkg.go.dev](https://pkg.go.dev/dario.cat/mergo) ·
[License: BSD-3-Clause](https://github.com/darccio/mergo/blob/master/LICENSE)

## Overview

Mergo merges two values of the same type by copying fields from a source into
a destination wherever the destination holds a zero value. It exists to remove
the pile of `if dst.Field == "" { dst.Field = src.Field }` statements that
accumulate in configuration and default-value code. First published in 2013 by
Dario Castañé, it is one of the oldest and most-depended-on utility packages in
the Go ecosystem, pulled in transitively by containerd, Docker/moby, Grafana
Loki, GoReleaser, and Kubernetes-adjacent tooling[^1].

The library is deliberately small and, as of v1.0.0, explicitly **frozen**: the
maintainer accepts bug fixes but no new features, deferring larger changes to a
hypothetical v2[^2]. That stability is the point — thousands of projects rely on
its exact behavior — but it also means the library's sharpest edge is permanent.
The core rule ("only overwrite zero-value fields") interacts badly with Go's
value semantics: a `false` bool, a `0` int, or an empty string is
indistinguishable from "unset", so those fields silently do not merge unless you
opt into override behavior. This is the single most common source of surprise in
mergo-using code.

A notable piece of trivia the README volunteers: the name is also a
[comune](http://en.wikipedia.org/wiki/Mergo) in the Italian province of Ancona.

## Getting Started

```
go get dario.cat/mergo
```

Note the import path is the **vanity URL** `dario.cat/mergo`, not
`github.com/darccio/mergo` and not the historical `github.com/imdario/mergo`.

```go
package main

import (
	"fmt"

	"dario.cat/mergo"
)

type Config struct {
	Host    string
	Port    int
	Verbose bool
}

func main() {
	defaults := Config{Host: "localhost", Port: 8080}
	cfg := Config{Host: "example.com"} // Port and Verbose are zero

	// Fill only the fields of cfg that are still at their zero value.
	if err := mergo.Merge(&cfg, defaults); err != nil {
		panic(err)
	}
	fmt.Printf("%+v\n", cfg) // {Host:example.com Port:8080 Verbose:false}
}
```

To let the source overwrite non-zero destination fields, pass `WithOverride`.
Merge is invoked as `mergo.Merge(&dst, src, opts...)`; the destination must be a
pointer.

## Architecture / How It Works

Mergo is built entirely on `reflect`. `Merge` walks the destination value field
by field: for structs it recurses into each **exported** field; for maps it
merges keys recursively. Unexported (private) fields are skipped because
reflection cannot set them, and there is no way around this short of the caller
exposing the fields. The recursion stops copying at a field only when the
destination there is non-zero (unless overridden).

Behavior is tuned through variadic option functions, the important ones being:

- `WithOverride` — source overwrites non-zero destination fields.
- `WithOverrideEmptySlice` / `WithAppendSlice` — control whether slices are
  replaced, treated-as-empty, or concatenated.
- `WithoutDereference` — when set, pointer fields are assigned as pointers rather
  than the library following them and merging the pointed-to values.
- `WithTransformers` — the escape hatch for types whose "zero value" is
  meaningless (see below).

The **transformer** interface is the mechanism for correctness on types that
don't follow the zero-value convention. The canonical example is `time.Time`: it
is a struct that is never `== zero` in the naive field sense, yet has an
`IsZero()` method. Without a transformer, mergo treats a populated `time.Time`
as "already set" and refuses to merge it. A user supplies a
`Transformer(reflect.Type) func(dst, src reflect.Value) error` that special-cases
`time.Time` and copies when `dst.IsZero()`. Any type with similar semantics
(decimals, `sql.Null*`, custom optionals) needs the same treatment.

`mergo.Map` provides the map↔struct direction: it can populate a struct from a
`map[string]interface{}` (matching capitalized keys to exported fields) or dump a
struct into a map. Mapping struct→map is intentionally **not recursive** — nested
struct members are assigned as values, not converted to nested maps.

## Production Notes

- **Zero-value ambiguity is the headline footgun.** Because mergo cannot tell
  "the user set this to `false`/`0`/`""`" apart from "unset", boolean and numeric
  config fields do not merge as many people expect. Teams that hit this either
  switch to pointer fields (`*bool`, `*int`) so nil means "unset", or restructure
  to merge in the opposite direction. Decide the convention before adopting it
  widely; retrofitting is painful.

- **`time.Time` and similar types silently misbehave** unless you register a
  transformer. This is the second most common bug report and is inherent to the
  design, not a fix-in-progress.

- **Structs nested inside maps are not merged**, because Go reflection cannot
  take the address of a map value to mutate it in place. The README states this
  as a hard limitation. If your data shape has `map[string]SomeStruct`, mergo
  will replace, not deep-merge, those entries.

- **The vanity-URL migration is a real supply-chain hazard.** v1.0.0 (2023) moved
  the canonical import path to `dario.cat/mergo`. Projects that still import
  `github.com/imdario/mergo` transitively, alongside code importing the new path,
  can end up with two copies of the package or build breaks. The maintainer's
  recommended remedy is a `replace github.com/imdario/mergo =>
  github.com/imdario/mergo v0.3.16` directive to pin the last old-path release[^2].

- **v0.3.9 shipped a regression** from a problematic PR; it was reverted in
  v0.3.10, which the maintainer calls stable but "not bug-free." Pin at or above
  0.3.10 if you are stuck on the 0.3.x line[^3].

- **Reflection cost.** Merge is not on any reasonable hot path — it is a
  startup/config operation — so its per-call reflection overhead rarely matters.
  Do not, however, call it per-request in a high-throughput loop; cache the merged
  result instead.

## When to Use / When Not

**Use when:**
- You assemble configuration from layers (defaults → file → env → flags) of the
  same Go struct type and want zero-value fields filled in.
- You want a battle-tested, frozen, widely-audited dependency with no churn.
- Your merge semantics fit "fill the blanks," optionally with `WithOverride`.

**Avoid when:**
- Your fields include meaningful `false`/`0`/`""` values that must survive a
  merge — the zero-value model will fight you.
- You need deep-merging of structs inside maps, or JSON-patch / strategic-merge
  semantics — mergo does not do these.
- You are primarily loading and layering config from files/env; a dedicated
  config library models precedence and types more directly than a generic struct
  merger.

## Alternatives

- knadh/koanf — layered configuration from files/env/flags with explicit
  precedence; use instead when the real goal is config loading, not struct merging.
- spf13/viper — the incumbent Go config library; use when you want an
  all-in-one config solution rather than a merge primitive.
- mitchellh/mapstructure — decodes `map[string]interface{}` into structs with
  hooks; use when your problem is really map→struct decoding.
- jinzhu/copier — copies fields between structs of different types; use when you
  need cross-type field copy rather than same-type zero-fill.
- google/go-cmp — comparison/diffing, not merging; use when you need to know what
  differs rather than to combine values.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-03 | First release as `github.com/imdario/mergo`. |
| 0.2.0 | 2015-04 | Behavior change around merging; pre-2015 users advised to re-verify[^3]. |
| 0.3.2 | 2016 | Added transformers via optional variadic argument (non-breaking)[^3]. |
| 0.3.9 | 2020-03-24 | Regression introduced by a bad PR[^3]. |
| 0.3.10 | 2020-07-18 | Reverted 0.3.9; go modules support. |
| 0.3.16 | 2023-05-27 | Last release under the `github.com/imdario/mergo` path. |
| 1.0.0 | 2023-06-20 | Vanity import path `dario.cat/mergo`; library frozen, no new features[^2]. |
| 1.0.1 | 2024-08-17 | Maintenance/bug fixes. |
| 1.0.2 | 2025-05-07 | Maintenance/bug fixes. |

## References

[^1]: "Mergo in the wild" and dependent lists — README and deps.dev.
    https://deps.dev/go/dario.cat%2Fmergo
[^2]: Mergo README, "Status" and "Important notes / 1.0.0" — vanity URL migration
    and frozen status. https://github.com/darccio/mergo#status
[^3]: Mergo README, "Important notes" — 0.3.9 regression, 0.3.2 transformer
    signature change, and the pre-April-2015 behavior note.
    https://github.com/darccio/mergo#important-notes

## Tags

go, golang, struct-merge, configuration, reflection, defaults, maps, utility-library, frozen, bsd-3-clause
