# davecgh/go-spew

> A deep pretty-printer for Go data structures, built for debugging — it will walk into unexported fields, follow pointers, and detect cycles.

[GitHub repo](https://github.com/davecgh/go-spew) ·
[pkg.go.dev](https://pkg.go.dev/github.com/davecgh/go-spew/spew) ·
[License: ISC](https://github.com/davecgh/go-spew/blob/master/LICENSE)

## Overview

go-spew is a single-purpose Go library that formats arbitrary Go values into a
verbose, human-readable dump: full type names, pointer addresses, map/slice
capacities, and a `hexdump -C`-style view for byte slices. Its distinguishing
trick is that it reports **unexported struct fields** — data that the standard
library's `fmt` `%#v` and `%+v` verbs deliberately omit — by reaching past
reflection's access rules with the `unsafe` package[^1]. For debugging opaque
third-party structs or runtime internals, this is the reason the package exists.

The library has effectively been feature-complete since 2016. The last tagged
release, v1.1.1, is from 2018[^2]; the `master` branch still receives occasional
commits but no new version. Despite that dormancy it remains one of the most
widely-imported packages in the Go ecosystem, almost entirely as a **transitive
dependency of stretchr/testify**, which uses spew to render values inside
assertion failure messages. Most projects that have go-spew in their `go.sum`
never call it directly.

The central tension: the same `unsafe`-based field access that makes spew useful
also makes it a liability in restricted environments and non-deterministic in
its default output. It is a debugging tool that leaked into millions of
production dependency graphs, and it should be understood in both roles.

## Getting Started

```bash
go get github.com/davecgh/go-spew/spew
```

```go
package main

import "github.com/davecgh/go-spew/spew"

type node struct {
	label string     // unexported — fmt would hide this
	next  *node
}

func main() {
	a := &node{label: "a"}
	a.next = a // deliberate cycle

	spew.Dump(a)
	// (*main.node)(0xc00008e000)({
	//  label: (string) (len=1) "a",
	//  next: (*main.node)(0xc00008e000)(<already shown>)
	// })

	// Deterministic variant for golden-file tests:
	cfg := &spew.ConfigState{SortKeys: true, DisablePointerAddresses: true}
	cfg.Dump(map[string]int{"b": 2, "a": 1})
}
```

`Dump` writes to stdout; `Fdump(w, ...)` targets any `io.Writer`; `Sdump(...)`
returns a string. The `Printf`/`Sprintf` family offers a compact inline style via
`%v`, `%+v` (adds addresses), `%#v` (adds types), and `%#+v` (both).

## Architecture / How It Works

Everything is driven by `reflect`. `dump` recurses over a `reflect.Value`,
switching on `Kind()` and emitting type-annotated output. Cycle detection is a
`map[uintptr]struct{}` of visited pointers; on a repeat it prints
`<already shown>` (or `<shown>` in the formatter) rather than looping forever.

The interesting part is unexported fields. `reflect` will let you *see* an
unexported field but not read its value through `Interface()`. spew works around
this in `bypass.go` by taking the field's address and constructing a new,
readable `reflect.Value` through `unsafe.Pointer`[^1]. This is the package's
whole reason for depending on `unsafe`, and it is gated behind build tags: the
`spew.go` implementation is compiled normally, while `bypasssafe.go` provides a
degraded fallback selected by the `safe` build tag, by GopherJS, or on Google
App Engine's classic runtime. In safe mode, unexported fields are simply not
dumped.

Behavior is configured through the `ConfigState` struct. The top-level functions
delegate to a package global, `spew.Config`; constructing your own `ConfigState`
gives an isolated, concurrently-usable configuration. Notable fields:

- `Indent` — per-level indent string (default one space; `"\t"` is common).
- `MaxDepth` — recursion cap; **unlimited by default**.
- `DisableMethods` — by default spew invokes `error` and `fmt.Stringer` methods;
  this turns that off.
- `DisablePointerMethods` — suppresses calling pointer-receiver `String()`/
  `Error()` on addressable non-pointer values (itself an `unsafe`-dependent path).
- `DisablePointerAddresses` / `DisableCapacities` — strip the two most common
  sources of non-deterministic output, for diffable test fixtures.
- `SortKeys` / `SpewKeys` — sort map keys (native types and Stringer/error types;
  `SpewKeys` is a last-resort string-spew fallback for exotic key types).

## Production Notes

- **Default output is non-deterministic.** Pointer addresses and map iteration
  order vary run to run. Anyone who has pasted `spew.Sdump` into a golden-file
  test has learned this the hard way. You need `SortKeys: true` *and*
  `DisablePointerAddresses: true` (and often `DisableCapacities: true`) before
  the output is stable enough to diff.
- **`MaxDepth` defaults to unlimited.** Cycles are handled, but a large acyclic
  graph (an ORM entity with all relations loaded, a parsed AST) can produce
  megabytes of output or stall a terminal. Set a depth cap when dumping unknown
  data.
- **Method invocation can surprise you.** Because `Stringer`/`error` methods run
  by default, spew may print a type's `String()` summary *instead of* the
  internal fields you were trying to inspect — and can trigger side effects or
  panics inside those methods. Set `DisableMethods` when you specifically want
  raw structure.
- **`unsafe` is a hard dependency for full fidelity.** In `safe`-tag builds,
  WASM/GopherJS, or hardened environments that forbid `unsafe`, unexported
  fields silently disappear from the dump. Don't rely on seeing private state in
  those targets.
- **It is a debugging aid, not a logger.** Reflection-heavy, verbose, and
  unstructured — never put `spew.Dump` on a hot path or in production log output.
  The README's own web example wraps output in `html.EscapeString` and warns
  against production use.
- **Maintenance is dormant.** No tagged release since v1.1.1 (2018); issues and
  PRs accumulate. This is largely fine — the API is stable and the problem is
  solved — but do not expect fixes, and pin the version you have. Security
  scanners occasionally flag the age and the `unsafe` usage; neither has
  translated into a known exploited vulnerability, but both are worth noting in
  audited codebases.

## When to Use / When Not

**Use when:**
- You need to inspect a struct's unexported fields during interactive debugging.
- You're dumping cyclic or deeply-nested data and want cycle-safe output.
- You want a readable hex + ASCII view of a `[]byte`.
- You're generating a stable, sorted textual snapshot of a value (with the
  determinism config applied).

**Avoid when:**
- You are comparing values in tests — reach for google/go-cmp's `cmp.Diff`
  instead of diffing two spew strings.
- You are in a build that forbids `unsafe` (you lose the main benefit).
- You need structured or production logging — use a real logger.
- Standard `fmt` `%+v`/`%#v` already tells you enough (zero dependency).

## Alternatives

- google/go-cmp — the correct tool for comparing structs in tests; `cmp.Diff`
  produces a focused diff instead of two full dumps you eyeball. Use when your
  actual goal is equality/assertion, not inspection.
- sanity-io/litter — dumps values as deterministic, valid-Go-source-like output;
  use when you want copy-pasteable, diff-stable snapshots out of the box.
- kr/pretty — lighter pretty-printer with a `Diff` helper; use when you want a
  smaller dependency and don't need unexported-field access.
- k0kubun/pp — colorized console pretty-printing; use for readable local
  debugging where terminal color helps more than pointer detail.
- Standard library `fmt` `%#v` / `%+v` — use when a zero-dependency, exported-
  fields-only view is sufficient.

## History

| Version | Date | Notes |
|---------|------|-------|
| (initial) | 2013-01-09 | Repository created; grew out of the Cyphertite project[^3]. |
| v1.0.0 | 2016-08-16 | First tagged release[^2]. |
| v1.1.0 | 2016-11-15 | `SortKeys`, capacity printing, config additions[^2]. |
| v1.1.1 | 2018-08-17 | Latest tagged release; API frozen since[^2]. |
| (untagged) | 2024-04-06 | Occasional `master` commits, no new release[^4]. |

## References

[^1]: go-spew source, `spew/bypass.go` — `unsafe`-based access to unexported
fields, with `bypasssafe.go` as the `safe`/GopherJS/App Engine fallback.
https://github.com/davecgh/go-spew/blob/master/spew/bypass.go
[^2]: go-spew releases and tags (v1.0.0 2016-08-16, v1.1.0 2016-11-15, v1.1.1
2018-08-17). https://github.com/davecgh/go-spew/tags
[^3]: "go-spew: a journey into dumping Go data structures" — original Cyphertite
blog post (archived). https://web.archive.org/web/20160304013555/https://blog.cyphertite.com/go-spew-a-journey-into-dumping-go-data-structures/
[^4]: Repository metadata — last push 2024-04-06, ISC license, ~6.4k stars as of
2026-07. https://github.com/davecgh/go-spew

## Tags

go, golang, debugging, pretty-printer, reflection, unsafe, developer-tools, testing, diagnostics, serialization
