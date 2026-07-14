# jqlang/jq

> A command-line JSON processor with its own functional, stream-oriented query language — `sed`/`awk` for JSON.

[GitHub repo](https://github.com/jqlang/jq) ·
[Official website](https://jqlang.org) ·
[License: MIT](https://github.com/jqlang/jq/blob/master/COPYING) (code; docs are CC-BY 3.0, bundled decNumber is ICU[^1])

## Overview

jq is a small C program that reads JSON on stdin, applies a *filter* written in its own language, and writes JSON on stdout. It was created by Stephen Dolan and first released in 2012[^2]. The pitch is narrow and durable: pipe JSON through `jq '.foo | .[] | select(.active)'` the same way you would pipe text through a chain of Unix tools. Zero runtime dependencies and a single portable binary made it a near-universal fixture in shell scripts, CI pipelines, and Dockerfiles.

The thing most newcomers underestimate is that jq is not a query syntax bolted onto a JSON parser — it is a full, Turing-complete functional language with backtracking, generators, `reduce`/`foreach`, path expressions, modules, and lexical closures. A jq "filter" is a function from one input value to a *stream* of zero or more output values, and almost every surprising behavior (a filter emitting nothing, or emitting many rows) follows from that stream model. This gives jq real expressive power but also a genuinely steep second half to the learning curve: `.name` is trivial, but `reduce`, `//`, `?//`, path tracking, and `--stream` are a different language to most users.

The other defining fact is maintenance history. After 1.6 (2018) the project went effectively dormant for roughly five years with no release. In 2022 maintenance was transferred to a new team under the `jqlang` GitHub organization, and 1.7 shipped in 2023[^3]. Releases have since resumed a regular cadence (1.8.x in 2025–2026). Anyone pinning behavior should know which era their jq comes from — the 1.6-to-1.7 gap means many installed jqs in the wild are years apart in features and bug fixes.

## Getting Started

```bash
# macOS / Linux / Windows
brew install jq            # Homebrew
apt-get install jq         # Debian/Ubuntu
docker run --rm -i ghcr.io/jqlang/jq:latest '.version' < package.json
```

```bash
# Filter, map, and select over an array of objects
echo '{"users":[{"name":"Tom","admin":true},{"name":"Brad","admin":false}]}' \
  | jq '.users[] | select(.admin) | .name'
# => "Tom"

# Reshape into new objects; -c for compact one-per-line output
curl -s https://api.example.com/repos \
  | jq -c '.[] | {name, stars: .stargazers_count}'

# Reduce a stream to a single value
jq -n '[1,2,3,4] | reduce .[] as $x (0; . + $x)'   # => 10
```

`-r` (raw output) strips quotes from string results — essential when feeding jq output back into shell. `-n` starts with `null` input; `-s` (slurp) reads the whole input stream into one array.

## Architecture / How It Works

jq compiles a filter program to bytecode and runs it on a small stack-based virtual machine[^4]. The core abstraction is the **stream**: a filter takes one input and produces a sequence of outputs. `empty` produces zero; `.[]` produces one per array element; the comma operator `a, b` concatenates streams. Pipes (`|`) feed each value from the left stream through the right filter. Execution is a **backtracking** model — when a downstream filter needs the next value, the VM backtracks into upstream generators to produce it, which is why jq can express Cartesian-product-like behavior without explicit loops.

Notable internals and language mechanics:

- **Path expressions** — `path(.a.b[])`, `getpath`, `setpath`, and the update operators (`|=`, `+=`) are built on jq's ability to track the *path* to a value, not just the value. This is what makes `.a.b |= ascii_upcase` work as an in-place update.
- **Regex** — provided by the bundled **Oniguruma** library (added in 1.5). `test`, `match`, `capture`, `sub`, `gsub` all route through it. Builds can use a system Oniguruma or the vendored copy (`--with-oniguruma=builtin`).
- **Numbers** — historically jq passed all numbers through IEEE 754 doubles, silently losing precision on large integers. 1.7 integrated **decNumber** to preserve literal number precision on decode/encode where possible, though arithmetic still generally goes through doubles[^3].
- **Modules** — jq has an `import`/`include` system and a search path (`-L`) for reusable `.jq` libraries, plus a standard library (`builtin.jq`) written partly in jq itself.
- **Streaming parser** — `--stream` emits `[path, leaf]` events instead of building the whole document in memory, the escape hatch for inputs too large to hold at once.

The build is autotools (`autoreconf`, `./configure`, `make`), with Oniguruma as a git submodule. The output is one static-linkable binary — no interpreter, no VM runtime to ship, which is the whole reason jq travels well.

## Production Notes

- **The `.` before a program string is the #1 gotcha.** `jq .foo` and `jq '.foo'` differ only when the shell mangles the unquoted form; always single-quote filter programs.
- **Whole-document memory by default.** Plain `jq` parses each input value fully into memory. Multi-gigabyte inputs need `--stream` (and a rewrite of the filter to consume `[path,leaf]` events), or splitting the input upstream. jq is not a low-memory streaming transformer unless you opt in.
- **Number precision is version-dependent.** Pre-1.7 jq will round-trip a 64-bit integer like `12345678901234567890` into a lossy double. 1.7+ preserves many literals but you cannot assume arbitrary-precision *arithmetic*. If exact large integers matter, test on your target version.
- **Behavior drift across the 1.6/1.7 boundary.** 1.7 changed and added many builtins, tightened some semantics, and fixed long-standing bugs. Scripts written against 1.6 (or the even older 1.5 shipped by long-term-support distros) may behave differently. Pin jq's version in CI rather than relying on whatever the base image provides.
- **Performance.** jq is fast enough for most shell-scale work but is not a high-throughput data engine; per-invocation startup and the interpreted VM make it slower than compiled reimplementations (gojq, jaq) on large workloads. For hot loops over big data, benchmark alternatives.
- **`-e` for exit codes.** In scripts, `--exit-status` makes jq return non-zero when the last output is `false`/`null`, so you can branch on jq results in shell conditionals instead of parsing stdout.
- **Error ergonomics are terse.** A filter that references a missing key emits `null`, not an error; a type mismatch (`.foo` on a string) *does* error. The `?` operator and `//` (alternative) are the standard tools for tolerating ragged data.

## When to Use / When Not

**Use when:**
- You need to slice, filter, reshape, or extract JSON in a shell script, Makefile, or CI step.
- You want a single dependency-free binary usable identically across dev machines, containers, and cloud runners.
- The transformation is non-trivial (joins across arrays, grouping, recursive descent) and a one-liner in another language would be longer.

**Avoid when:**
- Your data is genuinely large (multi-GB) and you need sustained streaming throughput — reach for a purpose-built tool or `--stream` with care.
- You need exact arbitrary-precision arithmetic on huge integers or decimals.
- The logic is complex enough that a real program (Python, a typed script) would be more maintainable than a growing wall of jq — jq programs get write-only quickly.
- You're processing YAML, TOML, or XML — use a tool built for that surface (though many wrap jq's language).

## Alternatives

- itchyny/gojq — pure-Go reimplementation of the jq language; more consistent error messages, embeddable as a Go library, no cgo. Use when you want jq semantics inside a Go program or a more predictable CLI.
- 01mf02/jaq — Rust reimplementation focused on speed and correctness; often faster on large inputs. Use when jq is a throughput bottleneck and you can tolerate minor language gaps.
- mikefarah/yq — YAML/XML/TOML processor with a jq-like (and jq-compatible) syntax. Use when your data is YAML rather than JSON.
- kislyuk/jq (pyjq) / JMESPath — embed JSON querying inside Python; JMESPath is a simpler, declarative spec used by the AWS CLI. Use when you want a library API rather than shelling out.
- fx / jless — interactive JSON viewers/explorers. Use when you're inspecting rather than transforming.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012-10 | Initial release by Stephen Dolan[^2]. |
| 1.5 | 2015-08 | Regex via Oniguruma, SQL-style operators, modules, `--stream`[^5]. |
| 1.6 | 2018-11 | Bugfix/feature release; then ~5 years dormant[^6]. |
| 1.7 | 2023-09 | New maintainer team under `jqlang` org; decNumber, many builtins, semantic fixes[^3]. |
| 1.7.1 | 2023-12 | Follow-up bugfix release. |
| 1.8.0 | 2025-06 | Continued maintenance under `jqlang`[^7]. |
| 1.8.2 | 2026-06 | Latest release as of this writing[^7]. |

## References

[^1]: jq `COPYING` — MIT for the code, CC-BY 3.0 for docs, ICU license for bundled decNumber. GitHub reports the repo license as NOASSERTION because of the mix. https://github.com/jqlang/jq/blob/master/COPYING
[^2]: Stephen Dolan, jq project origin. https://github.com/jqlang/jq
[^3]: jq 1.7 release notes. https://github.com/jqlang/jq/releases/tag/jq-1.7
[^4]: jq manual — filters, streams, and the language model. https://jqlang.org/manual/
[^5]: jq 1.5 release. https://github.com/jqlang/jq/releases/tag/jq-1.5
[^6]: jq 1.6 release. https://github.com/jqlang/jq/releases/tag/jq-1.6
[^7]: jq releases (1.8.x). https://github.com/jqlang/jq/releases

## Tags

json, cli, c, command-line-tool, data-processing, query-language, functional, unix, stream-processing, developer-tools
