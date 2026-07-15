# vmihailenco/msgpack

> Reflection-based MessagePack encoder/decoder for Go, with an `encoding/json`-shaped API.

[GitHub repo](https://github.com/vmihailenco/msgpack) ·
[Documentation](https://msgpack.uptrace.dev/) ·
[License: BSD-2-Clause](https://github.com/vmihailenco/msgpack/blob/v5/LICENSE)

## Overview

`vmihailenco/msgpack` implements the MessagePack binary serialization format[^1] for
Go. MessagePack is a compact, schema-less binary format with roughly the same data
model as JSON (nil, bool, int, float, string, binary, array, map) but smaller output
and faster parsing. This library is the reflection-based, ergonomic option in Go's
MessagePack space: `msgpack.Marshal` / `msgpack.Unmarshal` mirror `encoding/json`, and
struct behavior is driven by `msgpack:"..."` tags. If you already know Go's `json`
package, you know this API.

The defining tradeoff is convenience versus throughput. Because encoding walks Go
values with `reflect`, it is slower and allocates more than code-generation approaches
(notably tinylib/msgp), which emit type-specialized marshal methods at build time. For
most services — cache payloads, queue messages, RPC bodies — the reflection cost is
irrelevant next to the network and the ergonomics win. For hot paths measured in
millions of ops/sec, the codegen alternatives are materially faster.

The project is maintained by Vladimir Mihailenco (also author of go-redis and bun) under
the Uptrace umbrella. The current major version is **v5**, imported as
`github.com/vmihailenco/msgpack/v5` — omitting the `/v5` suffix is the most common
setup mistake and silently pulls the older, unmaintained v4 line. Development has been
quiet: the default branch is `v5` and the last push was mid-2024[^2], with roughly 50
open issues. It is stable and widely depended-on rather than actively evolving — treat
it as mature infrastructure, not a fast-moving project.

## Getting Started

```shell
go get github.com/vmihailenco/msgpack/v5
```

```go
package main

import (
	"fmt"

	"github.com/vmihailenco/msgpack/v5"
)

type Item struct {
	Foo string `msgpack:"foo"`
	Qux int    `msgpack:"qux,omitempty"` // dropped when zero
}

func main() {
	b, err := msgpack.Marshal(&Item{Foo: "bar"})
	if err != nil {
		panic(err)
	}

	var item Item
	if err := msgpack.Unmarshal(b, &item); err != nil {
		panic(err)
	}
	fmt.Println(item.Foo) // bar
}
```

For streaming and deterministic output, use the `Encoder` directly:

```go
enc := msgpack.NewEncoder(w)
enc.SetSortMapKeys(true)          // stable byte output for hashing/signing
enc.SetCustomStructTag("json")    // reuse existing json tags
```

## Architecture / How It Works

At its core the library is a reflection walker over Go values. `Marshal` opens an
`Encoder` on a byte buffer; `Unmarshal` opens a `Decoder`. Encoding dispatches on the
value's `reflect.Kind`, chooses the smallest MessagePack representation (e.g. an int
that fits in a byte becomes a `fixint`), and writes it. Decoding reads a format byte,
determines the wire type, and either fills a typed destination via reflection or, when
the destination is `interface{}`, materializes a concrete Go type.

Key extension points:

- **Struct tags** — `msgpack:"name"` renames a field, `,omitempty` skips zero values,
  `,alias:other` accepts an alternate name on decode, and `-` ignores a field.
- **CustomEncoder / CustomDecoder** — types can implement `EncodeMsgpack` /
  `DecodeMsgpack` to fully control their own wire form.
- **Extensions** — `RegisterExt` binds a Go type to a MessagePack ext type code, the
  mechanism the format reserves for application- or language-specific values.
- **Struct-as-array** — `SetUseArrayEncodedStructs(true)` (or a per-type `,as_array`
  tag) drops field names and encodes structs positionally, shrinking output at the cost
  of coupling both sides to field order.

By default structs encode as maps keyed by field name, which is self-describing and
tolerant of field reordering, but larger on the wire. `time.Time` is encoded using the
MessagePack timestamp extension. There is no schema and no code generation: the wire
format is derived entirely from Go types at runtime, which is what makes the API
pleasant and also what makes it slower than schema/codegen alternatives.

## Production Notes

**v4 → v5 is not a transparent bump.** The major version lives in the import path, so v4
and v5 coexist in a build. Beyond the path change, v5 adjusted default behaviors and the
concrete Go types produced when decoding into `interface{}`. Data written by one major
version and read by another can decode into different Go types (e.g. how untyped map keys
and integers surface), so pin one version across producers and consumers and test the
round-trip explicitly rather than assuming byte- and type-compatibility.

**Decoding into `interface{}` gives you the format's types, not yours.** MessagePack
picks the smallest integer encoding, so a value you wrote as `int` may come back as a
narrower Go type inside an `interface{}`. Type assertions like `v.(int64)` are fragile
across values; decode into concrete struct fields when you can, and be deliberate about
numeric handling on the generic path.

**Determinism is opt-in.** Map iteration order in Go is randomized, so by default the
same map can serialize to different byte sequences. If you hash, sign, or dedupe encoded
output, call `SetSortMapKeys(true)` — otherwise equal values produce unequal bytes.

**Cross-language interop needs care.** MessagePack is language-neutral, but the details
leak: `time.Time` rides on an ext code that another language's library must also
understand, struct-as-array output is positional and unlabeled, and `RegisterExt` codes
are a private contract between endpoints. Round-trips within Go are clean; round-trips to
Python/Ruby/JS libraries should be verified per type, not assumed.

**Throughput.** Reflection allocates. If profiling shows serialization is a bottleneck,
tinylib/msgp (code generation, near zero-allocation) is the usual escape hatch;
shamaton/msgpack is a faster reflection-based drop-in. For typical I/O-bound services the
difference does not show up.

**Maintenance cadence.** Low commit activity and an open-issue backlog mean fixes and new
features land slowly. This is fine for a stable format library, but do not expect rapid
turnaround on edge-case bugs — budget for a fork or a workaround if you hit one.

## When to Use / When Not

**Use when:**
- You want JSON-like ergonomics with smaller, faster binary output.
- You serialize Go structs for caches, message queues, or internal RPC where both ends
  are Go and you value convenience over maximum speed.
- You need custom encoders, extension types, or struct-tag-driven control.

**Avoid when:**
- Serialization is a measured hot path — prefer codegen (tinylib/msgp) or a schema format.
- You need a defined cross-language schema and forward/backward compatibility guarantees —
  Protobuf, FlatBuffers, or CBOR fit better.
- You want an actively, rapidly maintained library — this one is stable but slow-moving.

## Alternatives

- tinylib/msgp — code-generated MessagePack; use when you need maximum throughput and
  near-zero allocation and can run `go:generate`.
- shamaton/msgpack — reflection-based (with optional codegen) MessagePack that benchmarks
  faster; use when you want a quicker drop-in without changing the reflection workflow.
- ugorji/go — multi-format codec (MessagePack, CBOR, Binc, JSON) with codegen; use when
  one API must speak several wire formats.
- fxamacker/cbor — well-audited Go library for CBOR, an IETF-standard binary format; use
  when standardization and tooling matter more than the MessagePack ecosystem.
- protocolbuffers/protobuf — schema-first, cross-language, versioned wire contracts; use
  when you need an explicit schema rather than reflection over Go structs.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-10 | Repository created; early Go MessagePack implementation[^2]. |
| v4 | ~2019 | Go-modules-era line (`.../v4`); now unmaintained. |
| v5 | ~2021 | Current major line; import path `.../v5`, revised defaults and `interface{}` decoding, timestamp-ext time handling. |
| — | 2024-06 | Last pushed commit as of this writing; low ongoing activity[^2]. |

## References

[^1]: MessagePack specification. https://github.com/msgpack/msgpack/blob/master/spec.md
[^2]: GitHub repository metadata (created 2012-10-27, last push 2024-06-04, default branch `v5`, BSD-2-Clause). https://github.com/vmihailenco/msgpack

## Tags

go, golang, msgpack, messagepack, serialization, binary-format, encoding, reflection, marshaling
