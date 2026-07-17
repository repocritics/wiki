# mitchellh/copystructure

> Reflection-based deep copying of arbitrary Go values, including maps, slices, and pointers.

[GitHub repo](https://github.com/mitchellh/copystructure) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/github.com/mitchellh/copystructure) ·
[License: MIT](https://github.com/mitchellh/copystructure/blob/master/LICENSE)

## Overview

copystructure is a small Go library that produces a deep copy of a value at
runtime using reflection. Go has no built-in deep copy: assigning a struct or
passing it by value copies the top-level fields, but reference-typed members
(maps, slices, pointers, and the backing arrays behind them) still alias the
original. copystructure walks the value graph and reconstructs each reference
member so the result shares no mutable state with the input[^1].

It was written by Mitchell Hashimoto and belongs to the same cluster of
general-purpose reflection helpers as `mitchellh/mapstructure` and
`mitchellh/reflectwalk` — the latter is copystructure's direct dependency and
does the actual value traversal. The library became load-bearing infrastructure
mostly by transitive adoption: it appears deep in the dependency trees of the
HashiCorp stack (Terraform, Vault, Consul, Nomad) and anything built on
`go-cty`, where deep-copying dynamic configuration values is routine.

The defining tension is reflection itself. copystructure trades correctness for
speed: it will faithfully copy nearly any value shape you hand it, but it does
so by walking reflect trees at runtime, which is slow relative to hand-written
copy code and carries a set of edge-case limitations (unexported fields,
functions, channels, cycles) that are easy to trip over precisely because the
API looks like it "just works." The repository is now archived and read-only,
so its behavior is effectively frozen[^2].

## Getting Started

```
go get github.com/mitchellh/copystructure
```

```go
package main

import (
	"fmt"

	"github.com/mitchellh/copystructure"
)

func main() {
	orig := map[string]any{
		"tags": []string{"a", "b"},
	}

	dup, err := copystructure.Copy(orig)
	if err != nil {
		panic(err)
	}

	// Mutating the copy does not touch the original's slice.
	dup.(map[string]any)["tags"].([]string)[0] = "z"
	fmt.Println(orig["tags"]) // [a b]
	fmt.Println(dup.(map[string]any)["tags"]) // [z b]
}
```

`Copy` returns `interface{}`, so callers type-assert back to the concrete type.
For a struct value in, you get a struct value out; for a `*T` in, a fresh `*T`.

## Architecture / How It Works

The public surface is deliberately tiny: `Copy(v interface{}) (interface{}, error)`,
a `Config` struct for advanced control, and a `Copier` interface. Under the hood
`Copy` constructs a `reflectwalk` walker and traverses the value, allocating new
maps, slices, and pointer targets as it descends and reassembling them on the
way back up[^3].

Extension points, in order of how often they matter:

- **`Copier` interface** — any type implementing `Copy() (interface{}, error)`
  short-circuits reflection for itself. This is the escape hatch for types the
  generic walker cannot handle correctly (e.g. values guarding internal
  invariants, or types with unexported state that must be reconstructed).
- **`Config.Copiers` / `Config.PointerCopiers`** — a `map[reflect.Type]CopierFunc`
  registry that overrides copying for specific types without modifying those
  types. `time.Time` is a canonical case where callers register a custom copier.
- **`Config.ShallowCopiers`** — types listed here are copied by reference rather
  than descended into.
- **`Config.Lock`** — when true, values implementing `sync.Locker` are locked
  during their copy, guarding against concurrent mutation of the source while
  the walk is in progress.

Because copystructure builds on reflectwalk, its capabilities and its blind
spots are inherited from that library. The two are versioned and maintained
together; upgrading one without the other is a known source of subtle behavior
changes.

## Production Notes

- **Unexported fields are a hazard.** Reflection cannot set unexported struct
  fields through the normal `reflect` API, so generic deep copy of a struct with
  private state will not reproduce that state faithfully. If a type carries
  meaningful unexported fields, give it a `Copier` implementation rather than
  relying on the generic path — this is the most common correctness surprise.
- **Cycles are not tracked.** The walker does not memoize already-visited
  pointers, so a self-referential or cyclic pointer graph recurses without
  bound and will stack-overflow. Deep-copy only acyclic data with this library.
- **Functions and channels can't be deep-copied.** They are inherently
  reference-shaped; treat any copy of them as shared, not duplicated.
- **Performance.** This is runtime reflection over the whole value graph. It is
  fine for config objects and occasional copies; it is the wrong tool for a hot
  loop. If profiling points here, hand-write the copy or generate it. There is
  no way to make reflect-based traversal cheap.
- **Concurrency.** Without `Config.Lock`, copystructure makes no attempt to
  synchronize with concurrent writers of the source. Copying a value another
  goroutine is mutating is a data race regardless of the library.
- **Archived, so pin it.** The repo is archived and receives no fixes[^2]. The
  last tagged release is v1.2.0. Behavior will not change under you, but neither
  will bugs be fixed — vendor or pin the exact version and don't expect
  upstream response to issues.

## When to Use / When Not

**Use when:**
- You need a general deep copy of dynamically-shaped data (config trees,
  `map[string]any`, decoded values) and don't control every type.
- Copy frequency is low relative to the work around it.
- You're already in the HashiCorp / go-cty ecosystem where it's a transitive
  dependency anyway.

**Avoid when:**
- The copy sits in a hot path — reflection cost will dominate.
- Your values contain cycles, meaningful unexported fields, funcs, or channels
  without corresponding `Copier` implementations.
- You only need field-by-field struct copying between two named types — a
  name-matching copier or plain assignment is simpler and faster.

## Alternatives

- jinzhu/copier — copies fields between two structs by name; different problem
  (mapping, not faithful deep clone). Use when translating between DTO types.
- mohae/deepcopy — older reflection deep-copy with a similar single-function
  API. Use when you want a comparable approach without the HashiCorp lineage.
- barkimedes/go-deepcopy — reflection deep copy that explicitly handles cyclic
  structures. Use when your data has cycles copystructure would blow up on.
- encoding/gob or encoding/json round-trip — marshal then unmarshal to force a
  deep copy. Use for serializable data when you'd rather not add a dependency.
- Hand-written or code-generated copy methods — use when the value shape is
  fixed and known, and copy performance or unexported-field fidelity matters.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-06 | First commit; reflection deep copy built on reflectwalk[^1]. |
| v1.0.0 | 2019-08 | First tagged release under Go modules. |
| v1.1.x | 2020 | `Copier` interface and `Config` copier registries matured. |
| v1.2.0 | 2020-05 | Last tagged release; current pinned version across the ecosystem. |
| archived | — | Repository set read-only; no further changes[^2]. |

## References

[^1]: copystructure README and package overview. https://pkg.go.dev/github.com/mitchellh/copystructure
[^2]: Repository status: archived / read-only on GitHub (last push 2021-05-05). https://github.com/mitchellh/copystructure
[^3]: mitchellh/reflectwalk — the traversal library copystructure is built on. https://github.com/mitchellh/reflectwalk

## Tags

go, golang, deep-copy, reflection, clone, serialization, data-structures, hashicorp, utility, library
