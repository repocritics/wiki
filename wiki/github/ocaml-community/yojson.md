# ocaml-community/yojson

> The de facto standard JSON library of the OCaml ecosystem — a tree-model parser whose opam ubiquity far outweighs its GitHub star count.

[GitHub repo](https://github.com/ocaml-community/yojson) ·
[Official docs](https://ocaml-community.github.io/yojson/) ·
[License: BSD-3-Clause](https://github.com/ocaml-community/yojson/blob/master/LICENSE.md)

## Overview

Yojson parses JSON (and, via the companion `yojson-five` package, JSON5) into
an in-memory OCaml tree of polymorphic variants, and pretty-prints it back
out[^1]. It was written by Martin Jambon around 2010 as a faster replacement
for his earlier `json-wheel` library, and maintenance later moved to the
volunteer-run ocaml-community organization, where the repo lives today[^2].

Its 370 stars look small by JavaScript-ecosystem standards, but that number is
misleading: within opam, Yojson is one of the most-depended-upon packages, and
it is the substrate for the OCaml JSON codegen ecosystem — `ppx_deriving_yojson`,
Jane Street's `ppx_yojson_conv`, and Ahrefs' `atd` all target `Yojson.Safe.t`
as their wire type[^3]. Picking a JSON library in OCaml is mostly a decision
about whether you can afford *not* to use Yojson, since every ppx deriver
assumes it.

The defining tradeoff is the tree model itself. Yojson materializes the whole
document in memory as an association-list-based tree — simple to pattern-match,
trivially composable with OCaml's variant syntax, and fast enough for config
files and API payloads, but wrong for multi-hundred-megabyte documents or
incremental pipelines, which is `jsonm`'s territory.

## Getting Started

```bash
opam install yojson
# dune: (libraries yojson)
```

```ocaml
let json = Yojson.Safe.from_string {|
  {"number": 42,
   "string": "yes",
   "list": ["for", "sure", 42]}|}
(* val json : Yojson.Safe.t *)

let () = Format.printf "Parsed to %a" Yojson.Safe.pp json

(* Combinator-style access via the Util module *)
let n = Yojson.Basic.(
  from_string {|{"a": 1}|} |> Util.member "a" |> Util.to_int)
```

## Architecture / How It Works

Yojson ships three variant types generated from shared source (historically
via the `cppo` preprocessor):

1. **`Yojson.Basic.t`** — plain JSON: `` `Null ``, `` `Bool ``, `` `Int ``,
   `` `Float ``, `` `String ``, `` `Assoc `` (an association list), `` `List ``.
2. **`Yojson.Safe.t`** — adds `` `Intlit `` (integers too large for OCaml's
   native `int`, preserved as strings) plus non-standard `` `Tuple `` and
   `` `Variant `` extensions. "Safe" refers to integer safety, not memory
   safety; this is the type the ppx ecosystem standardized on.
3. **`Yojson.Raw.t`** — keeps all number literals as unparsed strings, for
   lossless round-tripping.

The lexer is hand-tuned `ocamllex`; there is no parser-generator stage and no
intermediate token list, which is why Yojson has historically been among the
faster OCaml JSON parsers. Parsing functions accept an optional reusable
buffer (`?buf`) to cut allocation in hot loops. Writers emit into a `Buffer`;
since 2.0.0 the pretty-printer is built on the stdlib `Format` module rather
than the old `easy-format` dependency[^4].

Each variant module carries a `Util` submodule of access combinators
(`member`, `to_string`, `to_int`, `filter_map`, ...) that raise `Type_error`
on shape mismatches — the idiomatic way to consume JSON without writing full
pattern matches. For concatenated or newline-delimited top-level values there
are `seq_from_string` / `seq_from_channel` readers, but note this streams
*between* documents, not *within* one: each value is still fully materialized.

## Production Notes

- **`?std` matters for interop.** By default the writers will emit Yojson's
  extensions: `` `Tuple ``/`` `Variant `` serialize to non-standard syntax and
  non-finite floats to `NaN`/`Infinity` literals that other parsers reject.
  Pass `~std:true` to force standard-compliant output (it raises on values
  with no JSON representation). Forgetting this is the classic Yojson bug
  found only when a Python or JS consumer chokes.
- **`Assoc` is an association list.** `Util.member` is O(n) per lookup and
  returns the first match; duplicate keys are preserved on parse and re-emitted
  on print. Repeated lookups into large objects deserve conversion to a real
  map first.
- **Basic vs Safe schism.** Libraries disagree on which type they speak;
  `Yojson.Safe.to_basic` bridges, but it is a full-tree copy and silently
  rewrites `` `Intlit ``/`` `Tuple ``/`` `Variant `` into plain JSON shapes.
- **Whole-document memory.** The tree costs a multiple of the raw text size.
  For log-scale files, switch to `jsonm` or process newline-delimited JSON
  with the `seq_*` readers.
- **Upgrade pains.** 2.0.0 (2022) dropped the `easy-format` and `biniou`
  dependencies; pretty-printed whitespace changed, which broke golden-file
  tests downstream[^4]. 3.0.0 (2025) is a cleanup release removing
  long-deprecated APIs[^5] — mechanical to absorb, but deps pinning old
  aliases need touching.
- **Maintenance cadence.** Volunteer-run with roughly one release a year and,
  as of mid-2026, no pushes since August 2025. Read this as feature-complete
  and stable rather than abandoned — but feature requests move slowly, and the
  README openly solicits help.

## When to Use / When Not

**Use when:**
- You are in OCaml and want JSON with zero ecosystem friction — every ppx
  deriver and most frameworks already speak `Yojson.Safe.t`.
- Payloads are config-file to API-response sized and fit comfortably in memory.
- You want derived converters: pair it with `ppx_deriving_yojson`,
  `ppx_yojson_conv`, or schema-first `atd`.

**Avoid when:**
- Documents are huge or arrive incrementally — a streaming decoder is the
  right shape.
- You need strict RFC 8259 behavior by default; Yojson's defaults are lenient
  and its extensions leak into output unless `~std:true` is passed.
- You are all-in on the Jane Street stack, where `jsonaf` integrates more
  naturally with `Core` and `Async`.

## Alternatives

- dbuenzli/jsonm — streaming, non-blocking decoder/encoder; use instead when
  documents are too large to hold in memory.
- mirage/ezjsonm — friendlier tree API layered over jsonm; use when you want
  jsonm's engine with less ceremony.
- janestreet/jsonaf — use when your codebase is already on Core/Async.
- ahrefs/atd — schema-first code generation (multi-language); use when the
  JSON contract should be a checked spec rather than hand-written converters.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2010–2019 | Martin Jambon era; `biniou`/`easy-format`-based printing. Final 1.x release 1.7.0 (2019-02)[^5]. |
| 2.0.0 | 2022-06 | Breaking: drops `easy-format`/`biniou` deps, pretty-printer rewritten on stdlib `Format`[^4]. |
| 2.1.0 | 2023-04 | Incremental additions on the 2.x line[^5]. |
| 2.2.0 | 2024-05 | JSON5 support lands via the separate `yojson-five` package[^5]. |
| 3.0.0 | 2025-05 | Cleanup major: removes long-deprecated APIs[^5]. |

## References

[^1]: Yojson API documentation on OCaml.org. https://ocaml.org/p/yojson/latest/doc/index.html
[^2]: ocaml-community/yojson repository. https://github.com/ocaml-community/yojson
[^3]: Yojson README, "Related tooling". https://github.com/ocaml-community/yojson#related-tooling
[^4]: Yojson changelog (CHANGES.md). https://github.com/ocaml-community/yojson/blob/master/CHANGES.md
[^5]: GitHub releases (version dates). https://github.com/ocaml-community/yojson/releases

## Tags

ocaml, json, json5, parser, serialization, tree-model, library, opam, data-interchange
