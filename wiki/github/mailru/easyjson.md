# mailru/easyjson

> Code-generated JSON marshalers for Go that trade a build step for 4-5x throughput over `encoding/json`.

[GitHub repo](https://github.com/mailru/easyjson) ·
[GoDoc](https://godoc.org/github.com/mailru/easyjson) ·
[License: MIT](https://github.com/mailru/easyjson/blob/master/LICENSE)

## Overview

easyjson is a Go code generator that emits type-specific `Marshal`/`Unmarshal`
functions for your structs, eliminating the runtime reflection that the standard
library's `encoding/json` relies on. You annotate a struct (or run with `-all`),
invoke the `easyjson` tool, and it writes a `<file>_easyjson.go` sibling
containing hand-optimized serialization code. The README reports 4-5x faster
unmarshaling and 3-4x faster marshaling than the standard library[^1]; the
approach is borrowed from and benchmarked against pquerna/ffjson, which pioneered
generated Go JSON codecs.

It was built at Mail.Ru (now VK), released in 2016[^2], and is one of the older
members of the "generated codec" cohort alongside ffjson and ugorji/go-codec. The
defining tradeoff is explicit: you accept a build-time code-generation step, a
committed generated file per package, and `go generate` friction in exchange for
throughput and much lower allocations on hot serialization paths. If your service
does not serialize JSON in a tight loop, that tradeoff rarely pays for itself.

As of 2026 the project is mature but slow-moving — roughly 4.9k stars and 460
forks, with commits arriving in bursts months apart rather than on a steady
cadence[^3]. The README still describes it as "early in its development," which
has been stale for years; in practice it is feature-frozen and maintained for
bug fixes rather than actively evolving. Modern alternatives (json-iterator,
goccy/go-json, and Go 1.24+'s revised `encoding/json`) have narrowed or closed
the gap it was built to exploit.

## Getting Started

```sh
# Go >= 1.17
go get github.com/mailru/easyjson && go install github.com/mailru/easyjson/...@latest
```

Annotate a struct and generate:

```go
// models.go
package models

//easyjson:json
type SomeStruct struct {
	Field1 string `json:"field1"`
	Field2 string `json:"field2,omitempty"`
}
```

```sh
easyjson -all models.go   # or drop -all and rely on //easyjson:json comments
```

Then serialize using the easyjson entry points (not `json.Marshal`, which would
route back through reflection):

```go
raw, err := easyjson.Marshal(&SomeStruct{Field1: "val1"})
var out SomeStruct
err = easyjson.Unmarshal(raw, &out)
```

The generated types also satisfy the standard `json.Marshaler`/`json.Unmarshaler`
interfaces, so `json.Marshal` still works — but at a performance penalty that
partly defeats the point[^1].

## Architecture / How It Works

The generator is itself bootstrapped: `easyjson` parses your file with Go's
`ast`/reflection tooling, writes a temporary Go program, and invokes `go run` on
it to emit the final marshaler code — the same bootstrapping trick ffjson uses.
This is why the tool historically required a working `GOPATH` and full Go build
environment, and why it cannot process `package main` files: the parser needs to
import your package, and `main` packages are not importable[^1].

Three internals matter for behavior:

- **`easyjson/jlexer`** — a hand-rolled JSON lexer used during unmarshaling. It
  does the minimal work to walk the input and skips over unmatched structures
  rather than fully validating the document, which is part of the speed story and
  part of the correctness caveat.
- **`easyjson/jwriter`** — a streaming-style writer the generated marshalers
  append into.
- **`easyjson/buffer`** — a chunked buffer pool (128 B up to 32 KB chunks; chunks
  ≥512 B are reused via `sync.Pool`) that keeps marshaling allocations low. Its
  defaults can be overridden via `buffer.Init()` before any serialization[^1].

Performance also depends on `unsafe`: on unmarshal, easyjson does zero-copy
`[]byte`→`string` conversions. This is opt-out via the `easyjson_nounsafe` build
tag, and is automatically disabled under the `appengine` build tag. The generator
exposes escape hatches — `nocopy` (point strings at the original buffer memory
for short-lived objects) and `intern` (deduplicate repeated string values) as
struct-tag options — plus an `easyjson/opt` package of pointer-free optional
wrappers for distinguishing missing from zero values.

## Production Notes

- **The generated file is a committed artifact.** `*_easyjson.go` must be checked
  in and regenerated whenever the struct changes. Forgetting to rerun the
  generator after a field edit produces silently stale serialization — a common
  footgun. Wire it into `//go:generate` and enforce a "no diff after generate"
  check in CI.
- **Object keys are case-sensitive**, unlike `encoding/json`, which matches keys
  case-insensitively. Migrating existing code onto easyjson can change decode
  behavior for inputs that relied on that leniency[^1].
- **Parsing is not validating.** The lexer skips over structure it does not need,
  so malformed JSON outside the fields you read may pass without error. Do not
  rely on unmarshal as an input-validation layer.
- **High-precision floats** are formatted at Go `strconv` default precision; if
  you need exact round-tripping of high-precision floating point, easyjson is the
  wrong tool[^1].
- **No true streaming.** The architecture assumes the full marshaled length is
  known before send, so there is no incremental streaming encode/decode path[^1].
- **`unsafe` in your dependency tree** can matter for security review, race
  detector runs, or platforms that forbid it. The `easyjson_nounsafe` tag exists
  but costs the zero-copy speedup.
- **Generics and newer Go features** are largely unaddressed; the generator
  predates them and the project's slow cadence means gaps close slowly, if at all.

## When to Use / When Not

**Use when:**
- You serialize the same set of structs on a genuinely hot path (high-QPS API,
  log/event pipelines) and have profiled `encoding/json` as a real cost.
- Allocation pressure and GC from reflection-based marshaling is measurable.
- A code-generation + committed-artifact workflow is acceptable in your build.

**Avoid when:**
- JSON serialization is not a bottleneck — the standard library is simpler and
  has closed much of the historical gap in recent Go releases.
- You want zero build steps or dislike committing generated code.
- You need strict validation, case-insensitive keys, high-precision floats, or
  streaming — behaviors easyjson deliberately trades away.
- You want a drop-in runtime library: json-iterator and goccy/go-json require no
  codegen and are easier to adopt.

## Alternatives

- json-iterator/go — drop-in `encoding/json`-compatible API, no codegen; the
  usual first stop when you want speed without a build step.
- goccy/go-json — reflection-based but heavily optimized; often competitive with
  generated codecs while staying a plain import.
- pquerna/ffjson — the codegen approach easyjson descends from; use it only for
  legacy parity, as it is effectively unmaintained.
- bytedance/sonic — JIT/SIMD-based decoder for amd64/arm64; pick it when you want
  peak throughput and can accept assembly-heavy internals and platform limits.
- Go standard `encoding/json` — the correct default; reach for a generator only
  after profiling proves it is the bottleneck.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-02-26 | First public release from Mail.Ru; ffjson-style codegen[^2]. |
| — | 2017-2020 | `nocopy`/`intern` tags, `opt` wrappers, buffer pool tuning added. |
| — | 2021+ | `go install ...@latest` install path for Go ≥ 1.17[^1]. |
| — | 2026-03 | Latest commit activity; maintained for fixes, feature-frozen[^3]. |

_easyjson does not publish semantic-version release tags in the conventional
sense; consumers typically pin to a commit or pseudo-version._

## References

[^1]: easyjson README — usage, options, limitations, benchmarks. https://github.com/mailru/easyjson/blob/master/README.md
[^2]: Repository creation date 2016-02-26, per GitHub API metadata. https://github.com/mailru/easyjson
[^3]: GitHub API repository metadata (stars, forks, last push 2026-03-14), retrieved 2026-07. https://api.github.com/repos/mailru/easyjson

## Tags

go, golang, json, serialization, code-generation, marshaling, performance, encoding, reflection-free, unsafe
