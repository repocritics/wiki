# goccy/go-json

> A drop-in replacement for Go's `encoding/json` that trades runtime-internal `unsafe` tricks for speed while keeping the standard library's exact API and semantics.

[GitHub repo](https://github.com/goccy/go-json) ·
License: MIT

## Overview

`go-json` is a JSON encoder/decoder for Go, written by Masaaki Goshima (goccy) and first published in 2020[^1]. Its stated goal is narrow and unusual for a performance library: be faster than `encoding/json` while remaining a byte-for-byte compatible replacement, so that adoption is a single import-path swap (`encoding/json` → `github.com/goccy/go-json`) with no code generation, no struct-tag changes, and no new interfaces to satisfy[^2].

That compatibility promise is the project's defining tension. Most fast JSON libraries buy their speed by giving something up — `easyjson` and `ffjson` require generated code, `gojay` asks you to hand-write encode/decode methods, `json-iterator/go` diverges from stdlib semantics in edge cases. `go-json` instead keeps the `encoding/json` surface and recovers the lost performance by reaching into Go runtime internals: it reads the unexported `rtype` layout, uses `typeptr` (the address of a type's descriptor) as a cache key, and compiles each type into an opcode sequence that a small virtual machine executes without per-field function calls or reflection[^2]. The result is fast, but the cost is coupling to compiler and runtime implementation details that are not part of Go's compatibility guarantee.

The library is a single-maintainer personal project and, as of this writing, still on a `v0.x` version line despite years of production use — a stated v1.0 has been the roadmap goal for some time but has not shipped[^1]. It is best known downstream as an optional JSON codec in the Gin web framework, selectable via the `go_json` build tag[^3].

## Getting Started

```bash
go get github.com/goccy/go-json
```

```go
package main

import (
	"fmt"

	"github.com/goccy/go-json"
)

type User struct {
	ID    int      `json:"id"`
	Name  string   `json:"name"`
	Roles []string `json:"roles,omitempty"`
}

func main() {
	b, _ := json.Marshal(User{ID: 1, Name: "ada", Roles: []string{"admin"}})
	fmt.Println(string(b)) // {"id":1,"name":"ada","roles":["admin"]}

	var u User
	_ = json.Unmarshal(b, &u)
	fmt.Printf("%+v\n", u)
}
```

The `json` package name and every exported symbol (`Marshal`, `Unmarshal`, `NewEncoder`, `NewDecoder`, `RawMessage`, `Marshaler`, `Number`, struct-tag grammar) match the standard library, so the change is normally invisible to existing code. Beyond parity it adds opt-in extras: JSON output colorization, propagating a `context.Context` into `MarshalJSON`/`UnmarshalJSON`, type-safe field filtering (`FieldQuery`), and `MarshalNoEscape` for callers that want to skip an allocation[^2].

## Architecture / How It Works

The core idea is to do the expensive per-type analysis once, cache it keyed by `typeptr`, and thereafter run a compiled program instead of walking `reflect` values.

- **typeptr dispatch.** From an `interface{}`, `go-json` reads the type-descriptor pointer directly (treating the interface as its two-word `{typ, ptr}` layout) and uses it as a cache key. Naively this is a `sync.Map` lookup; `go-json` goes further and, when the binary's total type information fits within a 2 MiB budget, uses the runtime's `typelinks` to build a flat slice indexed by `typeptr`, turning a map access into a bounds-free slice index. Above that budget it falls back to an `atomic`-based map[^2].
- **Opcode VM for encoding.** The first time a type is marshaled, `go-json` compiles it into a linked list of opcodes (`opStructFieldHead`, `opStructFieldInt`, `opStructEnd`, …) and executes them in one large `switch` loop, appending bytes as it goes. This avoids the per-field closure calls other libraries rely on. The opcode stream is then peephole-optimized — adjacent ops are fused (e.g. `opStructFieldHead` + `opStructFieldInt` → `opStructFieldHeadInt`) to cut the number of branches — and recursive types are handled by turning `CALL`-style recursion into `JMP`-style iteration with a manual value stack, a standard fast-VM technique[^2].
- **Bitmap field lookup for decoding.** Matching an input key to a struct field is the decoder hotspot. Instead of a map or hashed `switch`, `go-json` precomputes a bitmap (`[maxKeyLen][256]int8` for structs up to 8 fields, `int16` up to 16) so that each input byte narrows a running bit-mask; a field matches only if a bit survives to the key's end and the lengths agree. This is disabled for very long field names (~64+ bytes) to bound memory[^2].
- **NUL sentinel + bounds-check elimination.** The decoder appends a `\000` byte to the input buffer so end-of-input is caught by the same `switch` that dispatches on characters, avoiding a separate `cursor < len` compare on every byte; hot loops then read via `unsafe.Pointer` arithmetic to suppress the Go compiler's bounds checks[^2].

Everything above depends on the internal memory layout of `reflect.rtype` and on unexported `runtime`/`reflect` behavior (`typelinks`, interface word order, `unsafe` pointer arithmetic). None of that is covered by the Go 1 compatibility promise, which is the single most important fact to understand about this library.

## Production Notes

- **Go-version fragility is the headline risk.** Because `go-json` reads runtime-internal structures, a new Go release can change a layout or optimization and break it — historically the project has needed patches to keep working on new Go versions, and users on Go tip / prerelease toolchains have hit panics and mismatches before a fix landed. If you pin to the newest Go the day it ships, budget for the possibility that `go-json` needs a bump first.
- **`unsafe` means crashes, not just errors.** Correctness bugs in a reflection-based library surface as wrong output or an `error`; here they can surface as `panic`, memory corruption, or an incorrect result from `unsafe` misuse. The project maintains a separate fuzzing corpus (`go-json-fuzz`) precisely because this class of bug matters[^2]. Treat untrusted input with the same caution you would any `unsafe`-heavy dependency.
- **`MarshalNoEscape` is opt-in for a reason.** The zero-escape argument path was once the default but was demoted to an explicit `MarshalNoEscape` call after a Go compiler edge case where a large value that couldn't be stack-allocated failed to escape correctly[^2]. Don't reach for it unless the allocation shows up in a profile.
- **Compatibility is very good but not total.** The goal is stdlib parity and it is met far more completely than by `json-iterator/go` or `segmentio/encoding`, but "drop-in" is a target, not a proof. Behavior on exotic types, error message text, and streaming `Token` edge cases can differ; validate with your own round-trip tests before swapping in a service that depends on exact stdlib behavior.
- **The performance gap is narrowing.** Go's own `encoding/json` has improved over successive releases, and Go 1.24 introduced an experimental `encoding/json/v2` aimed at both speed and cleaner semantics. Re-benchmark on your workload and Go version rather than trusting the README's 2021-era charts; the win that justified `unsafe` in 2020 is smaller today for many payloads.
- **Single-maintainer, still `v0.x`.** No v1.0, no SemVer stability guarantee, and bus-factor of roughly one. Fine for a swappable codec behind a build tag (Gin's model); weigh it harder as a hard dependency deep in a system you can't easily revert.

## When to Use / When Not

**Use when:**
- You want measurable JSON throughput gains without adopting code generation or rewriting call sites.
- You already depend on stdlib `encoding/json` semantics and want to keep them while going faster.
- You're using Gin or another framework that exposes `go-json` behind a build tag, making it trivial to enable and revert.
- Your Go toolchain version is one you can hold stable and test against.

**Avoid when:**
- You track the latest Go release aggressively and can't tolerate a dependency lagging a runtime change.
- You process untrusted input in a high-assurance context and are unwilling to take on `unsafe`-based crash risk.
- You need a hard stability/support guarantee — the `v0.x`, single-maintainer status is a real risk for long-lived systems.
- Your JSON isn't a bottleneck; the stdlib (or `encoding/json/v2` on newer Go) is the lower-risk default.

## Alternatives

- encoding/json (Go standard library) — use instead when JSON isn't your bottleneck or you need the safest, guaranteed-stable option; on Go 1.24+ the experimental `encoding/json/v2` closes much of the speed gap.
- json-iterator/go — use when you want a faster stdlib-ish API and can accept semantic divergence in edge cases; broadly maintained but no longer very active.
- mailru/easyjson — use when you can run code generation and want top speed with no runtime `unsafe` reflection tricks.
- segmentio/encoding — use when encoding throughput dominates; strong encoder, but decoder API coverage (e.g. streaming `Token`) is partial.
- bytedance/sonic — use when you're on amd64/arm64 and want JIT/SIMD-backed speed and will accept its own runtime-internal coupling and platform constraints.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-04-19 | Repository created; opcode-based encoder, `unsafe`/`typeptr` design[^1]. |
| 0.9.x | 2021 | API surface expanded toward the stated v1.0 goal (per README roadmap)[^2]. |
| 0.10.0 | 2022 | Line that most production users have run for years; still pre-1.0. |
| 0.x (ongoing) | 2026-03 | Repository still receiving commits; v1.0 not yet released[^1]. |

## References

[^1]: goccy/go-json repository and metadata (created 2020-04-19; MIT; ~3.7k stars, last pushed 2026-03). https://github.com/goccy/go-json
[^2]: go-json README — features, compatibility comparison table, and "How it works" (buffer reuse, reflection elimination, opcode VM, typeptr→slice dispatch, bitmap field lookup, NUL sentinel). https://github.com/goccy/go-json/blob/master/README.md
[^3]: Gin framework — optional JSON codec selectable via the `go_json` build tag. https://github.com/gin-gonic/gin

## Tags

go, golang, json, serialization, encoding, decoder, encoder, performance, unsafe, drop-in-replacement, library
