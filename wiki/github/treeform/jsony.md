# treeform/jsony

> A loose, direct-to-object JSON parser and serializer for Nim — skips the
> intermediate node tree, customizes everything through hooks.

[GitHub repo](https://github.com/treeform/jsony) ·
[API reference](https://treeform.github.io/jsony) ·
[License: MIT](https://github.com/treeform/jsony/blob/master/LICENSE)

## Overview

jsony is a JSON library for the Nim programming language by treeform (Andre von
Houck), first tagged in January 2021[^1]. Its thesis is stated in the README's
opening line: "real world json is never what you want"[^2]. Instead of parsing
into an intermediate `JsonNode` tree and then mapping to your types — which is
what Nim's `std/json` + `to()` macro does — jsony generates type-specific
parsing code at compile time and deserializes byte buffers directly into your
objects. The same design applies in reverse for serialization. The author's own
benchmarks show roughly 2–6× faster serialization and ~3.5× faster
deserialization than `std/json`[^2], mostly from avoiding `JsonNode`
allocations, `StringStream` dispatch overhead, and per-number string
allocations.

The defining tradeoff is in the name "loose": extra JSON fields are silently
ignored, missing fields silently keep their default (zero) values, and
`snake_case` keys are automatically matched to Nim's `camelCase` fields[^2].
This makes jsony pleasant against messy third-party APIs and brutal for
validation — a typo'd field name produces a default-initialized value, not an
error. Strictness, when you need it, must be built by hand with hooks. Within
the small Nim ecosystem (~295 stars is upper-tier for a Nim library), jsony is
one of the standard choices for JSON, alongside `std/json` and status-im's
nim-json-serialization.

## Getting Started

```bash
nimble install jsony
```

```nim
import jsony

type User = object
  id: int
  displayName: string   # matches both "displayName" and "display_name"

# JSON -> object: extra fields ignored, missing fields default
let u = """{"id": 1, "display_name": "Tom", "extra": true}""".fromJson(User)
doAssert u.displayName == "Tom"

# object -> JSON
doAssert u.toJson() == """{"id":1,"displayName":"Tom"}"""
```

The library has no dependencies beyond the Nim standard library[^2].

## Architecture / How It Works

jsony is a compile-time code generator, not a runtime reflection system. Calls
like `fromJson(T)` instantiate generic `parseHook` procs for `T` and every
nested type, producing a specialized parser that reads directly from the input
string with a moving index — no tokenizer object, no DOM, no streams. Numbers
are parsed straight out of the memory buffer, bypassing the allocations that
`parseInt`/`$` would create[^2].

Customization is a single mechanism: overloadable hook procs that the generated
code picks up via Nim's static dispatch[^2]:

- `parseHook` / `dumpHook` — full control over parsing/serializing any type
  (e.g. `DateTime` as an ISO string, an id-keyed JSON object as a `seq`).
- `newHook` — pre-populate defaults before fields are parsed.
- `postHook` — fix up an object after parsing (sanitize, migrate versions).
- `enumHook` — map wire enum strings ("RED") to Nim enum names (`c2Red`).
- `renameHook` — remap field names at parse time (e.g. JSON `type` → `kind`,
  since `type` is a Nim keyword).
- `skipHook` — omit fields (passwords, connections) from serialization.

Because dispatch is static, hooks must be exported (`proc parseHook*`) and
visible at the call site that instantiates the parser — a hook defined but not
exported is silently not used[^2].

Coverage is broad for a small library: seqs, arrays, tuples, options, enums,
`Table`/`OrderedTable`, sets, ref objects, and two features that matter in
practice: full case-variant object support (if the discriminator field is not
first, the parser scans ahead to find it, then rewinds and parses normally) and
`RawJson`, a `distinct string` that defers parsing of a subtree entirely —
useful for store-and-forward payloads[^2]. `toStaticJson` serializes `const`
values at compile time.

## Production Notes

- **Looseness is the footgun.** There is no strict mode. Required-field
  validation, unknown-field rejection, and schema checking do not exist; a
  renamed upstream field degrades silently to default values. Teams that need
  contract enforcement layer `postHook` assertions on top, or use
  status-im/nim-json-serialization which was designed with strictness options.
- **Unexported hooks fail silently.** The most common jsony bug report shape:
  a `parseHook` without `*` compiles fine and is simply ignored. There is no
  warning.
- **Compile-time cost.** Every distinct type instantiates its own parser and
  serializer; large object graphs increase compile times and binary size
  relative to a DOM-based approach. Generic-heavy code can also produce the
  usual opaque Nim instantiation errors when a nested type has no handler.
- **Maintenance profile.** Single-maintainer treeform ecosystem library.
  Release cadence is sparse — 1.1.5 (Feb 2023) to 1.1.6 (Nov 2025) was nearly
  three years[^3] — but the repo is not dead: last push May 2026, and the
  library is small and stable enough that gaps are mostly benign. 36 open
  issues/PRs sit unresolved, consistent with hobby-pace triage.
- **Benchmarks are the author's own.** The 2–6× numbers come from the README's
  microbenchmark against four Nim alternatives[^2]; treat them as directional,
  and re-measure on your payloads (deeply nested vs. flat, string-heavy vs.
  numeric) before citing them.
- **Case-variant rewind.** Discriminator-not-first objects cost an extra scan
  pass over the object body; on hot paths, put the discriminator first in the
  JSON if you control the producer.

## When to Use / When Not

**Use when:**
- You are consuming third-party or evolving JSON APIs in Nim and want messy
  input mapped onto clean typed objects with minimal ceremony.
- Serialization throughput matters (game servers, ETL, IPC) and `std/json`'s
  node tree shows up in your profiles.
- You need per-type escape hatches (dates, id-keyed maps, keyword field names)
  — the hook system handles all of these locally.

**Avoid when:**
- Silent acceptance of malformed input is unacceptable (payments, config
  validation, security-sensitive parsing) — jsony's looseness works against
  you, and you will reimplement strictness by hand.
- You need to inspect unknown/dynamic JSON structure — that is `std/json`'s
  DOM job (though jsony can embed `JsonNode` for json-in-json cases).
- You are not writing Nim; this library is Nim-only by construction (its speed
  comes from Nim's compile-time generics).

## Alternatives

- nim-lang/Nim `std/json` — use instead when you need DOM-style traversal of
  unknown JSON or want zero third-party dependencies; slower, stricter typing
  via `to()`.
- status-im/nim-json-serialization — use instead when you need configurable
  strictness and flavors (it backs Nimbus, an Ethereum client); heavier and
  slower per jsony's benchmarks[^2].
- planetis-m/eminim — use instead if you prefer streaming deserialization via
  `std/streams`; less feature coverage.
- disruptek/jason — compile-time JSON serialization experiments;
  serialize-only, more curiosity than production tool.
- treeform/flatty — same author, use instead when both ends are Nim and you
  want a compact binary format rather than interoperable JSON.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.1 | 2021-01-03 | First tagged release[^1]. |
| 1.0.0 | 2021-03-04 | Stable API[^3]. |
| 1.1.0 | 2021-11-04 | 1.1 line begins[^3]. |
| 1.1.3 | 2022-01-03 | Fixes; steady maintenance period[^3]. |
| 1.1.5 | 2023-02-09 | Last release before a ~33-month gap[^3]. |
| 1.1.6 | 2025-11-02 | Maintenance release after the gap; repo active into 2026[^3]. |

## References

[^1]: jsony releases, v0.0.1 — 2021-01-03. https://github.com/treeform/jsony/releases/tag/0.0.1
[^2]: jsony README (features, hooks, benchmark tables). https://github.com/treeform/jsony#readme
[^3]: jsony releases page (all tag dates). https://github.com/treeform/jsony/releases

## Tags

nim, json, serialization, parsing, deserialization, hooks, compile-time, performance, library
