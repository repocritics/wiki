# mitchellh/mapstructure

> Reflection-based Go library for decoding generic `map[string]interface{}` values into typed structs, and back.

[GitHub repo](https://github.com/mitchellh/mapstructure) ·
[License: MIT](https://github.com/mitchellh/mapstructure/blob/main/LICENSE)

## Overview

mapstructure fills a gap the Go standard library leaves open. `encoding/json`
and friends decode bytes directly into a struct, but that assumes you know the
target shape before you read the data. mapstructure works one level up: it takes
an already-decoded `map[string]interface{}` (or any Go value) and reflects it
onto a struct, so you can inspect a discriminator field first — a `"type"` key,
a version number — and then decode the rest into the concrete type you chose[^1].

That indirection made it the connective tissue of the HashiCorp configuration
stack. Terraform, Vault, Consul, Nomad, and Packer all parse HCL or JSON into
generic maps and then land them onto typed structs with mapstructure; Viper, the
most widely used Go config library, uses it as its decode layer[^2]. The
`mapstructure:"..."` struct tag is one of the most common non-stdlib tags in the
Go ecosystem for that reason.

The defining tension is that the repository is **archived and read-only** as of
2024[^3]. mitchellh (Mitchell Hashimoto) archived most of his personal Go
libraries after leaving HashiCorp. The code is stable and correct, but no bug
fixes, security patches, or Go-version compatibility updates will land here. The
maintained continuation is `go-viper/mapstructure`, a drop-in fork run by the
Viper maintainers; new projects should depend on that instead (see Alternatives).

## Getting Started

```
$ go get github.com/mitchellh/mapstructure
```

```go
package main

import (
	"fmt"
	"github.com/mitchellh/mapstructure"
)

type Server struct {
	Name    string `mapstructure:"name"`
	Port    int    `mapstructure:"port"`
	Tags    []string
	Extra   map[string]interface{} `mapstructure:",remain"`
}

func main() {
	input := map[string]interface{}{
		"name":  "web-01",
		"port":  8080,
		"tags":  []string{"prod", "edge"},
		"owner": "sre", // unmapped → captured by ,remain
	}

	var s Server
	if err := mapstructure.Decode(input, &s); err != nil {
		panic(err)
	}
	fmt.Printf("%+v\n", s) // {Name:web-01 Port:8080 Tags:[prod edge] Extra:map[owner:sre]}
}
```

For anything beyond the trivial case, construct a `Decoder` from a
`DecoderConfig` rather than calling the top-level `Decode`, because the config is
where all the useful behavior lives.

## Architecture / How It Works

Everything runs through `reflect`. `Decode(input, &output)` is a thin wrapper
that builds a default `DecoderConfig` and calls `Decoder.Decode`. The decoder
walks the target struct's fields, matches each to a key in the source map
(case-insensitive by default), and recursively assigns values, converting kinds
as it goes. There is no code generation and no schema — the struct's reflected
type *is* the schema.

The real surface area is `DecoderConfig`:

- **`DecodeHook`** — a function invoked on every value before assignment. This is
  the primary extension point. Built-in hooks handle common coercions:
  `StringToTimeDurationHookFunc` (`"30s"` → `time.Duration`),
  `StringToSliceHookFunc`, `StringToTimeHookFunc`, and `TextUnmarshallerHookFunc`.
  `ComposeDecodeHookFunc` chains several together. Nearly every non-trivial user
  of the library writes at least one custom hook.
- **`WeaklyTypedInput`** — enables loose coercion: `"true"` → `bool`, `42` →
  `"42"`, empty values → zero values, single values → single-element slices. Off
  by default; Viper turns it on, which is why Viper feels forgiving about types.
- **`ErrorUnused` / `ErrorUnset`** — turn silently-ignored extra keys, or unset
  target fields, into hard errors. Essential for strict config validation and
  off by default (the default is permissive).
- **`Metadata`** — populates a `Metadata` struct listing which keys were used and
  which were left unused, without erroring.
- **`TagName`** — defaults to `"mapstructure"` but can be repointed (some
  projects reuse `"json"`).

Two struct-tag options carry most of the ergonomic weight: `,squash` inlines an
embedded struct's fields into the parent (flattening nested config), and
`,remain` captures every unmatched key into a catch-all map. Errors accumulate
into a single `*mapstructure.Error` that reports *all* decode failures at once
rather than stopping at the first — a deliberate choice for surfacing config
mistakes in bulk.

Decoding is bidirectional: pass a struct as input and a `*map[string]interface{}`
as output and the same machinery runs in reverse.

## Production Notes

- **Reflection cost is real.** mapstructure is meaningfully slower and more
  allocation-heavy than `encoding/json` into a concrete type, because it reflects
  per-field on every call with no cached plan. Fine for config loaded once at
  startup (its design center); a poor fit for a hot decode path handling many
  requests per second. If it shows up in a profile, that is the signal to switch
  to generated code or direct unmarshalling.
- **Case-insensitive matching by default.** Field matching ignores case, so
  `Port`, `port`, and `PORT` all bind to the same field. This is convenient for
  config but can mask typos and produce surprising collisions; there is no option
  to make matching strict-case.
- **Silent drops without `ErrorUnused`.** By default, keys in the source map with
  no matching struct field are discarded silently. A misspelled config key
  becomes a zero-valued field with no warning. Set `ErrorUnused: true` (or check
  `Metadata.Unused`) in any config parser you expect users to typo in.
- **`WeaklyTypedInput` hides bugs.** It is convenient but coerces aggressively —
  a string where you expected a number will not fail, it will parse. Prefer strict
  typing plus explicit hooks unless you specifically want the leniency.
- **Archived: pin and plan a migration.** Because the repo is frozen, treat the
  last tag (v1.5.0) as terminal. Newer Go releases have not broken it so far, but
  any future incompatibility will not be fixed upstream. `go-viper/mapstructure`
  is API-compatible; migration is largely a find-and-replace of the import path,
  and it is where security-relevant fixes now land.
- **`,squash` with conflicting names is undefined-ish.** Flattening two embedded
  structs that both expose a field of the same name resolves in field order and
  can silently pick the wrong one; keep squashed field names disjoint.

## When to Use / When Not

**Use when:**
- You parse config or dynamic data into a generic map first, then need it typed.
- You need a discriminator field to pick the target struct at runtime.
- You want rich decode-time coercion (durations, custom types) via hooks.
- You are already in the Viper / HashiCorp-style config world.

**Avoid when:**
- You control the wire format and can decode bytes straight into a struct — use
  `encoding/json` and skip the intermediate map entirely.
- The decode path is hot; reflection overhead will show.
- You are starting fresh — depend on `go-viper/mapstructure`, not this archived
  original.

## Alternatives

- go-viper/mapstructure — the maintained drop-in fork; use instead of this repo for any new or actively developed code.
- mitchellh/hashstructure — same author, when you need a hash of a struct rather than a decode into one.
- knadh/koanf — use when you want a lighter, more modular config loader than Viper that still leans on mapstructure-style decoding.
- go-playground/validator — use alongside decoding when the real need is struct validation, not map-to-struct conversion.
- encoding/json (stdlib) — use when you own the format and can unmarshal bytes directly into a known struct.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-05-20 | First commit; extracted for HashiCorp config parsing[^1]. |
| v1.0.0 | 2018-11 | First tagged module-era release. |
| v1.4.x | 2020–2021 | `,remain`, decode-hook additions, metadata refinements. |
| v1.5.0 | 2023-01 | Final release on this repo. |
| archived | 2024 | Repository set read-only; maintenance moves to go-viper fork[^3]. |

## References

[^1]: mapstructure README — "decoding generic map values to structures and vice versa." https://github.com/mitchellh/mapstructure
[^2]: Viper uses mapstructure for its `Unmarshal` decode layer. https://github.com/spf13/viper
[^3]: Repository is marked archived (read-only) on GitHub; the maintained continuation is https://github.com/go-viper/mapstructure

## Tags

go, golang, reflection, decoding, configuration, struct-mapping, serialization, hashicorp, viper, archived
