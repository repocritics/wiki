# json-iterator/go

> A reflection-based drop-in replacement for Go's `encoding/json`, faster in its era — now archived and read-only.

[GitHub repo](https://github.com/json-iterator/go) ·
[Official website](http://jsoniter.com/migrate-from-go-std.html) ·
[License: MIT](https://github.com/json-iterator/go/blob/master/LICENSE)

## Overview

jsoniter (json-iterator for Go) is a JSON encoder/decoder that presents the same
API surface as the standard library's `encoding/json` and claims 100%
compatibility, while running faster and allocating less on typical payloads[^1].
Created by Tao Wen (taowen) in 2016, it became one of the most-depended-on JSON
libraries in the Go ecosystem: Kubernetes (`k8s.io/apimachinery`) and Prometheus
both pulled it in during the late-2010s as a transparent throughput win over the
stdlib, which is a large part of how it accumulated ~13.9k stars.

The defining trick is that jsoniter is still reflection-based — it does not
require code generation like `mailru/easyjson` — but it builds and caches a
per-type "codec" the first time it sees a struct, so subsequent
marshal/unmarshal calls skip most reflection overhead. It leans on
`modern-go/reflect2` to touch reflected values without the allocations the
standard `reflect` package incurs. The result was a genuine 2–6× decode speedup
against Go 1.8-era `encoding/json`[^1].

The central tension today is that jsoniter is **archived** (read-only since
May 2024[^2]) and its performance lead has eroded. The stdlib itself got faster
across Go 1.12–1.21, and a newer generation of libraries — `goccy/go-json`,
`bytedance/sonic` — now beats jsoniter while remaining maintained. jsoniter
also never delivered on "100% compatible": several edge cases diverge from
`encoding/json`, which matters precisely because the library is sold as a
transparent substitute.

## Getting Started

```bash
go get github.com/json-iterator/go
```

The migration is a single import swap plus a package-level variable, so existing
`json.Marshal` / `json.Unmarshal` call sites are untouched:

```go
package main

import (
	"fmt"

	jsoniter "github.com/json-iterator/go"
)

// Drop-in: matches encoding/json behavior (HTML escaping, key ordering, etc.)
var json = jsoniter.ConfigCompatibleWithStandardLibrary

type User struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

func main() {
	b, _ := json.Marshal(&User{ID: 1, Name: "Tom"})
	fmt.Println(string(b)) // {"id":1,"name":"Tom"}

	var u User
	_ = json.Unmarshal([]byte(`{"id":2,"name":"Brad"}`), &u)
	fmt.Printf("%+v\n", u) // {ID:2 Name:Brad}
}
```

## Architecture / How It Works

jsoniter is organized around three pieces:

1. **Config** — a `Config` struct (`ConfigDefault`, `ConfigCompatibleWithStandardLibrary`,
   `ConfigFastest`, or a custom one) that fixes behavior knobs at construction:
   HTML escaping, `SortMapKeys`, `UseNumber`, indentation, float precision. Each
   frozen config owns its own codec cache.
2. **Codec cache** — the first encode/decode of a concrete type walks it with
   reflection and compiles a `ValEncoder` / `ValDecoder` tree, keyed by
   `reflect.Type`. Later calls reuse the cached codec, so steady-state cost is
   the codec tree walk, not reflection.
3. **Streaming core** — `Stream` (writer) and `Iterator` (reader) operate on
   byte buffers with minimal intermediate allocation. `reflect2` is used to read
   and write struct fields via unsafe pointer offsets rather than
   `reflect.Value` method calls, which is where most of the allocation savings
   come from.

Because field access goes through `unsafe`-backed pointer arithmetic, jsoniter's
correctness depends on assumptions about Go's memory layout and the `reflect`
internals. This is a legitimate speed source and a legitimate risk surface — it
is the reason a JSON library needs updating as Go's runtime evolves, and the
reason an archived one accrues latent risk over time.

The public API deliberately mirrors `encoding/json` (`Marshal`, `Unmarshal`,
`NewEncoder`, `NewDecoder`, `RawMessage`, `Marshaler`/`Unmarshaler`
interfaces), plus a lower-level `Iterator`/`Stream` API for hand-tuned parsing
that the stdlib does not expose.

## Production Notes

**It is archived.** As of May 2024 the repository is read-only[^2]: no fixes,
no Go-version compatibility updates, no security patches. This is the single
most important operational fact. New code should not adopt it; existing users
should plan a migration.

**"100% compatible" is aspirational, not literal.** Known divergences from
`encoding/json` over the years include differences in number/float formatting,
handling of `json.RawMessage`, error message text, and some malformed-input
behavior. If your system relies on byte-exact output or on specific stdlib error
strings, verify against your own payloads before swapping — the drop-in promise
can break golden-file tests and downstream parsers silently.

**`ConfigFastest` is a footgun.** It buys speed partly by lowering float
marshaling precision (roughly 6 significant digits) and by not sorting map keys.
For metrics, money, or anything where numeric fidelity or deterministic output
matters, `ConfigFastest` will change values. Use
`ConfigCompatibleWithStandardLibrary` unless you have measured the tradeoff.

**The performance edge has narrowed.** The headline benchmark on the project
page compares against a Go 1.8-era stdlib[^1]. Modern `encoding/json` closed much
of the gap, and `goccy/go-json` / `bytedance/sonic` now typically outrun
jsoniter. "Always benchmark with your own workload" — the README's own caveat —
is the right stance; do not adopt jsoniter for speed without measuring against
current alternatives on real data.

**Concurrency and caching.** Configs are safe for concurrent use once frozen;
the codec cache is populated lazily and guarded internally. The first call for
each new type pays a compilation cost, so latency-sensitive services see a warmup
tail on cold types.

## When to Use / When Not

**Use when:**
- You are maintaining an existing service already built on jsoniter and a rewrite
  is not justified — it still works on current Go versions in practice.
- A transitive dependency (older Kubernetes/Prometheus code) pulls it in and you
  just need to understand what it does.

**Avoid when:**
- You are starting anything new — pick a maintained library.
- You need byte-exact `encoding/json` compatibility for golden tests or wire
  contracts.
- You want maximum speed — `sonic` (x86-64, JIT/SIMD) or `goccy/go-json` are
  faster and maintained.
- You need long-term security/maintenance guarantees; an archived repo provides
  neither.

## Alternatives

- goccy/go-json — reflection-based drop-in replacement, faster than jsoniter and
  actively maintained; use when you want jsoniter's transparency without the
  archive risk.
- bytedance/sonic — JIT/SIMD JSON for amd64; use when raw throughput on x86-64
  servers is the priority and you can accept architecture-specific behavior.
- mailru/easyjson — code-generated marshalers, no runtime reflection; use when you
  control the types and want the lowest allocation via `go generate`.
- valyala/fastjson — schemaless parsing without struct binding; use when you only
  need to pluck a few fields out of large/dynamic documents.
- encoding/json (stdlib) — the baseline; with the experimental `encoding/json/v2`
  landing in recent Go, use when maintenance and correctness outweigh peak speed.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-11-30 | Repository created; drop-in `encoding/json` replacement[^3]. |
| adoption | 2017–2018 | Pulled into Kubernetes and Prometheus for throughput. |
| v1.1.x | 2019–2021 | Semantic-import-versioned line; `reflect2`-backed codec path matured. |
| v1.1.12 | 2021-08 | One of the last tagged releases. |
| archived | 2024-05-27 | Last push; repository set read-only[^2]. |

## References

[^1]: json-iterator/go README — benchmark table and "100% compatible drop-in replacement" claim. https://github.com/json-iterator/go
[^2]: GitHub repository metadata (`archived: true`, `pushed_at: 2024-05-27`), retrieved 2026-07. https://github.com/json-iterator/go
[^3]: GitHub repository metadata (`created_at: 2016-11-30`). https://api.github.com/repos/json-iterator/go

## Tags

go, golang, json, serialization, deserialization, json-parser, encoding, reflection, drop-in-replacement, archived, performance
