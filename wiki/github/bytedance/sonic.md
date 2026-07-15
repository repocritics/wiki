# bytedance/sonic

> A JIT- and SIMD-accelerated JSON library for Go that trades portability and RFC strictness for throughput.

[GitHub repo](https://github.com/bytedance/sonic) ·
[License: Apache-2.0](https://github.com/bytedance/sonic/blob/main/LICENSE)

## Overview

Sonic is ByteDance's drop-in replacement for Go's standard `encoding/json`,
open-sourced in 2021[^1]. Its premise is that reflection-driven JSON on Go's
standard library is a bottleneck at ByteDance scale, and that a serializer which
compiles per-type machine code at runtime (JIT) plus hand-written SIMD kernels
can process large and small payloads several times faster. The public API mirrors
`encoding/json` — `sonic.Marshal`, `sonic.Unmarshal` — so migration is usually a
one-line import swap.

The defining tension is performance versus portability and safety. Sonic reaches
into `unsafe`, ships assembly generated from C, and uses `//go:linkname` to bind
against Go runtime internals. That is why it supports only AMD64 and ARM64 (ARM64
requires Go 1.20+), a bounded Go version range (currently 1.18–1.26, with Go 1.24.0
specifically broken without a build flag), and falls back to `encoding/json` on
any unsupported environment[^2]. It is a fast path for the platforms it targets,
not a universal library.

A second tension is standards conformance. By default sonic does **not** HTML-escape
output and does **not** sort map keys — both are performance choices that diverge
from `encoding/json` and, per the project's own README, from RFC 8259[^3]. Teams
that depend on byte-identical output must opt into `ConfigStd`.

## Getting Started

```bash
go get github.com/bytedance/sonic
```

```go
package main

import (
	"fmt"
	"github.com/bytedance/sonic"
)

type User struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}

func main() {
	out, _ := sonic.Marshal(&User{Name: "Ada", Age: 36})
	fmt.Println(string(out)) // {"name":"Ada","age":36}

	var u User
	_ = sonic.Unmarshal(out, &u)
	fmt.Printf("%+v\n", u)   // {Name:Ada Age:36}
}
```

For latency-sensitive services, warm the JIT for known types at startup so the
first request doesn't pay compilation cost (see Production Notes):

```go
import "reflect"
import "github.com/bytedance/sonic"

func init() {
	sonic.Pretouch(reflect.TypeOf(User{}))
}
```

## Architecture / How It Works

Sonic has three layers that account for its speed and its constraints:

- **JIT-compiled codecs.** On the first `Marshal`/`Unmarshal` of a concrete type,
  sonic reflects over the type once and compiles a specialized encoder/decoder to
  machine code, caching it per type. Subsequent calls skip reflection entirely.
  The assembler is `golang-asm` (a fork of Go's internal assembler), which the
  README notes is not ideal for runtime compilation — hence the warmup story.
- **SIMD kernels.** Low-level scanning (whitespace skipping, string/number parsing,
  validation, escaping) is done by vectorized routines written in C and shipped as
  Go-compatible assembly. This is the source of the AMD64/ARM64-only support: the
  kernels must exist for the target instruction set.
- **`ast.Node` — a lazy JSON AST.** For read/modify workloads sonic offers a
  self-contained AST that parses lazily: `sonic.Get(data, "a", 1, "b")` walks only
  the path you ask for instead of unmarshaling the whole document. `Index()` uses
  byte offsets and is faster than `Get()`'s key scan. Nodes can be mutated
  (`Set`/`Unset`) and re-serialized. `ast.Node` is not safe for concurrent reads
  unless you pass the `ConcurrentRead` search option, because of the lazy-load design.

The coupling that matters is to the Go runtime itself. Sonic's `linkname` bindings
and assembly assume a specific runtime layout, so each new Go release is a
compatibility event: Go 1.24.0 broke sonic and requires `-ldflags="-checklinkname=0"`
until a fixed toolchain[^2]. Upgrading Go and upgrading sonic are effectively
linked decisions.

## Production Notes

- **JIT warmup can cause first-hit latency spikes or OOM.** Compiling a codec for a
  huge or deeply nested schema happens on the first request touching that type. The
  README explicitly warns this can cause request timeouts or process OOM, and
  recommends `Pretouch`/`PretouchMany` at startup for large schemas or latency-
  sensitive services, with `WithCompileRecursiveDepth` for very deep types.
- **Zero-copy strings retain the input buffer.** When decoding string values with no
  escape sequences, sonic references them directly into the original JSON buffer
  instead of copying. This is a CPU win but keeps the entire input alive as long as
  any decoded object is retained — measured at 20–80% extra memory in ByteDance's
  own tests. For cached results, use the `CopyReturn` search option or copy strings
  yourself.
- **Output is not stdlib-identical by default.** No HTML escaping, no sorted keys.
  Downstream consumers that hash or diff JSON (signatures, ETags, zstd dictionaries)
  will see different bytes than `encoding/json`. Use `ConfigStd`, or opt into
  `encoder.EscapeHTML` and `encoder.SortMapKeys` individually — each costs ~10–15%.
- **Silent fallback masks "no speedup".** On an unsupported OS/arch/Go version, sonic
  transparently degrades to `encoding/json`. Your code keeps working but the
  performance you deployed for silently disappears — verify the platform in CI, don't
  assume the JIT path is active.
- **`unsafe` + generated assembly is an audit surface.** Security-sensitive or
  regulated codebases should weigh that sonic parses untrusted input in hand-written
  vectorized assembly with `linkname` hooks into the runtime. Fuzzing exists upstream,
  but the trust profile differs from a pure-Go parser.

## When to Use / When Not

**Use when:**
- You are on Linux/macOS/Windows AMD64 or ARM64, on a supported Go version, and JSON
  ser/de is a measured hot path.
- You need partial reads/edits of large JSON without full unmarshaling (`ast.Node`).
- You can pin and co-manage Go and sonic versions together.

**Avoid when:**
- You target architectures beyond AMD64/ARM64 (386, WASM, riscv, mips) — you get the
  `encoding/json` fallback and no benefit.
- You require byte-for-byte RFC 8259 / stdlib-identical output by default.
- You want minimal `unsafe`/assembly surface, or a library that tracks new Go releases
  without a version-compatibility dance.

## Alternatives

- goccy/go-json — pure-Go fast JSON, near-drop-in for `encoding/json`; use when you
  want most of the speed without JIT, assembly, or arch restrictions.
- json-iterator/go — mature reflection-based accelerator; use when broad portability
  and a long track record matter more than peak throughput.
- tidwall/gjson + tidwall/sjson — read/modify JSON by path without structs; use when
  you only query or patch documents, similar to sonic's `ast.Node` but pure Go.
- mailru/easyjson — compile-time code generation instead of runtime JIT; use when you
  can codegen and want no runtime compilation cost or `unsafe` at startup.
- encoding/json (stdlib) — the baseline; use when correctness, portability, and zero
  dependencies outweigh speed, or Go 1.24+'s revised stdlib encoder is fast enough.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-05 | Repository open-sourced by ByteDance; AMD64-only, JIT + SIMD[^1]. |
| v1.6.0 | 2022 | `decoder.MismatchTypeError` added — reports type mismatch but keeps decoding[^3]. |
| — | 2022+ | ARM64 support added (requires Go 1.20+)[^2]. |
| — | 2026-06 | Active; supports Go 1.18–1.26, with Go 1.24.0 excluded pending a runtime fix[^2]. |

## References

[^1]: bytedance/sonic repository and README — "A blazingly fast JSON serializing & deserializing library, accelerated by JIT and SIMD." https://github.com/bytedance/sonic
[^2]: sonic README, "Requirement" — Go 1.18–1.26, Go 1.24.0 unsupported per golang/go#71672, AMD64/(ARM64 on Go 1.20+); fallback to `encoding/json` off supported environments. https://github.com/bytedance/sonic/blob/main/README.md
[^3]: sonic README, "Usage" / "Print Error" — default divergence from `encoding/json` HTML escaping and key sorting (not RFC 8259 conformant); `MismatchTypeError` introduced in v1.6.0. https://datatracker.ietf.org/doc/html/rfc8259

## Tags

go, json, serialization, jit, simd, high-performance, bytedance, encoding, parser, unsafe
