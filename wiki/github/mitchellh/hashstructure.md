# mitchellh/hashstructure

> Deterministic uint64 hashes of arbitrary Go values via reflection — a cache-key and change-detection primitive, not a cryptographic digest.

[GitHub repo](https://github.com/mitchellh/hashstructure) ·
[Godoc](https://pkg.go.dev/github.com/mitchellh/hashstructure/v2) ·
[License: MIT](https://github.com/mitchellh/hashstructure/blob/master/LICENSE)

## Overview

hashstructure walks an arbitrary Go value with `reflect` and folds it into a
single `uint64`. The intended use is comparing complex values cheaply: key a
map or set on a struct, detect that config changed without a field-by-field
diff, or de-duplicate objects locally instead of shipping them over the
network. It is a small, single-purpose library — one exported function
(`Hash`) plus a handful of struct-tag and option knobs.

The library was written by Mitchell Hashimoto (co-founder of HashiCorp) and
saw heavy internal use across HashiCorp tooling, which shaped its priorities:
stability of the hash for a given value, ignore/set field semantics, and
pluggable hashers. It is *not* a content-addressing or security primitive. The
default hasher is FNV-1a (non-cryptographic), collisions are possible, and the
output is not stable across the v1/v2 format boundary. Treat the hash as an
in-process equality shortcut, never as an identity you persist or trust against
an adversary.

The defining tension is reflection: hashstructure trades raw speed and
compile-time safety for the ability to hash *anything* without writing per-type
code. For hot paths over known types a hand-written or code-generated hash will
beat it; hashstructure earns its place when the types are many, deeply nested,
or not known until runtime.

## Getting Started

```bash
go get github.com/mitchellh/hashstructure/v2
```

```go
import "github.com/mitchellh/hashstructure/v2"

type Config struct {
    Name     string
    Replicas int
    Secret   string   `hash:"ignore"` // excluded from the hash
    Tags     []string `hash:"set"`    // order-insensitive
}

a := Config{Name: "web", Replicas: 3, Secret: "x", Tags: []string{"a", "b"}}
b := Config{Name: "web", Replicas: 3, Secret: "y", Tags: []string{"b", "a"}}

ha, _ := hashstructure.Hash(a, hashstructure.FormatV2, nil)
hb, _ := hashstructure.Hash(b, hashstructure.FormatV2, nil)
// ha == hb → true: Secret is ignored, Tags compared as a set
```

## Architecture / How It Works

`Hash(v, format, opts)` recursively reflects over `v`. Scalars are written
directly into a `hash.Hash64`; composite kinds are folded structurally:

- **Structs** hash each exported field, mixing in the field name so two
  structs with the same values in different fields don't collide.
- **Maps** XOR the per-entry hashes so map iteration order (which Go
  randomizes) doesn't change the result.
- **Slices/arrays** hash element-by-element in order — unless the field is
  tagged `hash:"set"`, in which case elements are combined order-insensitively.
- **Pointers** are followed; `nil` has a defined encoding.

Behavior is steered three ways. **Struct tags** under the `hash:` key:
`ignore` drops a field, `set` gives a slice set semantics, `string` hashes the
result of the value's `String()` method (the documented way to hash
`time.Time` meaningfully). **`HashOptions`** carries a custom `Hasher`
(`hash.Hash64`), an alternate `TagName`, and flags like `ZeroNil` and
`IgnoreZeroValue` that control whether nil/zero fields participate. **The
`Hashable` interface** lets a type override the whole process by supplying its
own hash, bypassing reflection entirely.

The `format` argument is the load-bearing detail. `FormatV1` is the original
algorithm; `FormatV2` is a distinct scheme that fixed collision-prone cases in
v1. They produce different numbers for the same input, and the v2 module path
(`/v2`) is a separate import from v1. Passing `FormatV1` to the v2 package
reproduces the old hashes when you must stay compatible with previously stored
values.

## Production Notes

- **Reflection cost is real.** Every `Hash` call re-walks the value with
  `reflect`. For high-frequency hashing of known types, benchmark against a
  hand-rolled hasher; hashstructure is convenience-first, not throughput-first.
- **Not collision-free, not cryptographic.** The default FNV-1a hasher is fast
  and non-cryptographic. Collisions are possible and, under a chosen-input
  attacker, cheap to construct. Never use the output as a security token,
  content address, or dedup key where an adversary controls inputs.
- **The v1 collision caveat is explicit.** The maintainer recommends v2: v1 had
  "significant hash collision issues" that depend on the shape of your data.
  HashiCorp ran v1 for years without incident — the risk is real but
  data-dependent, so don't assume v1 is safe just because it hasn't bitten you.
- **Hashes are not portable.** Values are unstable across the v1↔v2 boundary,
  across option changes (`ZeroNil`, custom hasher), and are not guaranteed
  stable across Go versions or architectures. Don't persist them or compare
  them across services — recompute and compare within a single run.
- **Only exported fields are hashed.** Unexported struct fields are skipped,
  which can silently make two "different" values hash equal; values behind
  `interface{}` are hashed by their concrete dynamic type.
- **Archived, read-only.** The repository was archived by the author and last
  saw a commit on 2023-01-03. There is no active maintenance, no new releases,
  and open issues will not be triaged. It is stable and widely vendored, but
  any bug you find is yours to work around or fork.

## When to Use / When Not

**Use when:**
- You need a cheap in-process equality/change-detection key for complex,
  nested, or runtime-unknown Go values.
- You want ignore/set field semantics without writing custom comparison code.
- You're already in the HashiCorp/Go ecosystem and want a well-worn primitive.

**Avoid when:**
- You need a cryptographic or adversary-resistant digest (use `crypto/sha256`
  over a canonical serialization).
- You need hashes that are stable across languages, services, or persisted over
  time.
- You're hashing a small set of known hot-path types where reflection overhead
  matters — code-gen or hand-write instead.

## Alternatives

- cnf/structhash — reflection-based struct hashing that emits md5/sha1 digests;
  use when you want a cryptographic-style hash string rather than a `uint64`.
- fxamacker/cbor (or `encoding/json` canonicalization) + crypto/sha256 — use
  when you need a deterministic, cross-language, persistable content hash.
- mitchellh/mapstructure — unrelated in purpose (decodes maps into structs) but
  frequently confused with it; reach for it when the task is decoding, not
  hashing.
- google/go-cmp — use when you only need to compare two in-memory values for
  equality and never actually need a hash key.
- mitchellh/copystructure — companion library for deep-copying arbitrary Go
  values via the same reflection style; pairs with, rather than replaces,
  hashstructure.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-01-07 | First commit; reflection-based `Hash` for arbitrary values.[^1] |
| v1.x | 2016–2019 | Original format (`FormatV1`); heavy internal HashiCorp use. |
| v2.0.0 | ~2020 | New module path `/v2`, `FormatV2` fixing v1 collision cases.[^2] |
| last commit | 2023-01-03 | Final change before the repo went dormant.[^1] |
| archived | — | Repository set read-only by the author; no further releases. |

## References

[^1]: mitchellh/hashstructure repository and commit history. https://github.com/mitchellh/hashstructure
[^2]: Package README, "Note on v2" — recommends v2 to avoid v1 collisions; `FormatV1` retained for backward compatibility. https://github.com/mitchellh/hashstructure/blob/master/README.md

## Tags

go, golang, hashing, reflection, struct-hash, cache-key, change-detection, fnv, deterministic, archived, hashicorp
