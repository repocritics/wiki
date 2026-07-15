# TomWright/dasel

> One CLI and Go library that queries and mutates JSON, YAML, TOML, XML, CSV, INI, HCL and KDL with a single selector syntax.

[GitHub repo](https://github.com/TomWright/dasel) ·
[Official website](https://daseldocs.tomwright.me) ·
[License: MIT](https://github.com/TomWright/dasel/blob/master/LICENSE)

## Overview

Dasel ("data-select") is a Go command-line tool, first released in 2020[^1], that
does for arbitrary structured data what `jq` does for JSON: read a value, update
it, or convert the whole document to another format. Its distinguishing bet is
format-neutrality — the same selector expression works against JSON, YAML, TOML,
XML, CSV, INI, HCL and KDL, and the format is decided by flags (`-i`, `-o`) rather
than by which tool you reached for. In a world where teams already juggle `jq`,
`yq`, and format-specific parsers, dasel's pitch is "learn one syntax, point it at
anything." As of 2026 it sits around 8k stars and is listed in Awesome Go[^2].

The project has been through two hard breaks in its life. The v1 selector syntax
was replaced in v2, and v3 replaced it again with a genuine expression language —
a lexer/parser producing an evaluable AST, with functions (`each()`, `search()`),
recursive descent (`..`), and comparison operators. This is dasel's defining
tension: the v3 query language is far more capable than early dasel or than `yq`'s
path syntax, but it is also idiosyncratic — not jq-compatible, not JSONPath, not
JMESPath — so knowledge is non-transferable and each major version has invalidated
existing scripts and muscle memory[^3].

Dasel is maintained primarily by a single author (Tom Wright). It is actively
worked on — commits land through mid-2026 — but bus-factor and the historically
churny query syntax are the two facts to weigh before standardizing a fleet on it.

## Getting Started

```sh
brew install dasel
# or:
go install github.com/tomwright/dasel/v3/cmd/dasel@master
```

```sh
# Read a nested value
echo '{"foo": {"bar": "baz"}}' | dasel -i json 'foo.bar'
# "baz"

# Modify in place; --root prints the whole document, not just the touched node
echo '{"foo": {"bar": "baz"}}' | dasel -i json --root 'foo.bar = "bong"'

# Convert JSON to YAML — same tool, different output codec
cat data.json | dasel -i json -o yaml

# Transform every element
echo '[1,2,3]' | dasel -i json --root 'each($this = $this*2)'   # [2,4,6]

# Find anywhere in the tree
echo '{"a":{"bar":"baz"}}' | dasel -i json 'search(bar == "baz")'
```

## Architecture / How It Works

Dasel is a single Go binary with no runtime dependencies, distributed as prebuilt
binaries, a Homebrew formula, and a `go install` target. Internally it is three
layers: a set of per-format **readers/writers** (each format is a codec that
marshals to and from a common in-memory node model), a **parser** that turns a
selector string into an expression AST, and an **evaluator** that walks the AST
against the node tree.

The v3 rewrite is the architecturally significant change. Earlier dasel treated a
selector as a path of dot/bracket steps. v3 introduces a real expression grammar —
tokens, literals, functions, binary operators, `$this`/variable references — so a
selector like `each($this = $this*2)` or `search(name == "x")` is parsed and
evaluated rather than pattern-matched. That is what unlocks in-place arithmetic,
conditional search, and recursive descent, but it also means the selector language
is now large enough to have its own learning curve and its own documentation site.

The format-neutral node model is the source of both the value and the friction.
Because every format normalizes to the same internal representation, cross-format
conversion is close to free — but the representations do not line up cleanly.
XML has attributes and ordered mixed content; CSV is a flat table with no nesting;
TOML and YAML disagree on how comments, key ordering, and typed scalars survive a
round-trip. Dasel papers over these differences with a lowest-common-denominator
model, so a JSON→YAML→JSON round-trip is usually faithful while a JSON→CSV→JSON one
is lossy by construction. The abstraction is the product, and its leaks are the
format edges.

## Production Notes

- **Query syntax is version-locked.** Scripts written for v1 do not run on v2, and
  v2 selectors do not run on v3. If you pin dasel in CI or shell scripts, pin the
  major version and read the migration notes before bumping. Treat a dasel upgrade
  as a code change, not a patch.
- **`go install ...@master` tracks the development tip**, not a release. For
  reproducible builds use a tagged version or a pinned binary; `@master` can change
  behavior between two `go install` runs.
- **Round-trip fidelity varies by format.** Comments, key ordering, anchors/aliases
  (YAML), and XML attribute vs. element distinctions are the usual casualties. If
  you need a config file edited *in place with comments preserved*, test that
  specific format first — do not assume it. For YAML specifically this is a common
  disappointment relative to `yq`, which was built around comment preservation.
- **`--root` vs. default output is a frequent footgun.** By default dasel prints
  only the evaluated selector's result; to emit the modified whole document you must
  pass `--root`. Modify-in-a-pipeline scripts that forget it silently emit a
  fragment.
- **Not a streaming processor.** Dasel loads the document into memory and evaluates
  against the tree. It is not designed for gigabyte-scale or line-delimited
  streaming workloads the way `jq --stream` targets; for very large inputs it will
  hold the whole structure in RAM.
- **Single-maintainer cadence.** Response time on issues and the pace of new format
  support depend largely on one person. This is fine for a stable CLI utility; weigh
  it if dasel becomes load-bearing infrastructure.

## When to Use / When Not

**Use when:**
- You routinely touch multiple config formats and want one selector syntax instead
  of `jq` + `yq` + ad-hoc parsers.
- You need quick format conversion (JSON↔YAML↔TOML) in shell scripts.
- You want a dependency-free single binary for CI, containers, or Makefiles.
- You need to *modify* values in structured files, not just read them.

**Avoid when:**
- Your team already knows `jq` deeply and works only in JSON — the switching cost
  buys little.
- You need guaranteed comment/formatting preservation on YAML round-trips.
- You need streaming over very large inputs.
- You want a query language with transferable, standardized knowledge (JSONPath,
  JMESPath) rather than a dasel-specific grammar.

## Alternatives

- jqlang/jq — the reference JSON processor; richer, standardized-ish language, but JSON-only. Use when you live in JSON and want the deepest tooling.
- mikefarah/yq — YAML-first (also JSON/XML/props), built around comment/format preservation. Use when faithful in-place YAML edits matter.
- kislyuk/yq — thin `jq` wrapper for YAML/XML; use when you want jq's exact language over YAML.
- jmespath/jp — JMESPath CLI; use when you want a standardized, portable query spec across languages.
- itchyny/gojq — pure-Go jq reimplementation; use when you want jq semantics embeddable in a Go program.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2020-09-22 | First release; unified selector CLI over JSON/YAML/TOML/XML[^1]. |
| v1.x | 2020–2021 | Dot/bracket path selectors; multi-format read + write. |
| v2.0 | 2021 | Selector syntax overhaul; standardized flags and behavior[^3]. |
| v3.0 | 2024 | Expression-language rewrite: AST evaluator, functions, `search()`, recursive descent, KDL support. |
| — | 2026-06 | Actively maintained on `master`; ~8k stars[^2]. |

## References

[^1]: TomWright/dasel — repository, created 2020-09-22. https://github.com/TomWright/dasel
[^2]: Awesome Go listing for dasel. https://github.com/avelino/awesome-go
[^3]: Dasel documentation — selectors and migration between major versions. https://daseldocs.tomwright.me

## Tags

go, golang, cli, json, yaml, toml, xml, config, data-transformation, query-language, devops-tools, parser
