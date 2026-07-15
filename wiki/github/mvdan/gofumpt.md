# mvdan/gofumpt

> A stricter superset of gofmt: the same guarantees, a few more opinions, and no configuration.

[GitHub repo](https://github.com/mvdan/gofumpt) ·
[Go reference](https://pkg.go.dev/mvdan.cc/gofumpt) ·
[License: BSD-3-Clause](https://github.com/mvdan/gofumpt/blob/master/LICENSE)

## Overview

gofumpt is a Go source formatter written by Daniel Martí (`mvdan`), maintained since 2019[^1]. It is a fork of the standard library's `cmd/gofmt` that adds a fixed set of extra formatting rules on top of what `gofmt` already enforces. The design constraint is deliberate and load-bearing: gofumpt only ever formats a *subset* of what `gofmt` accepts, so running `gofmt` after `gofumpt` produces no changes. It extends the canonical formatter rather than competing with it[^2].

The defining property is the absence of choice. Like `gofmt`, gofumpt takes no style configuration — there is no config file, and the only behavioral flag is `-extra` for two additional opt-in rules. This is the same philosophy that makes `gofmt` uncontroversial in Go teams, pushed one notch further: gofumpt collapses stylistic debates (empty lines around function bodies, `0o755` vs `0755`, grouping of `std` imports, short vs long var declarations) into a single tool with a single answer. The tension is that gofumpt's answers are one maintainer's opinions, not a Go team standard, so adopting it is a team decision that gofmt never forced you to make.

Because it is a hard fork of `gofmt`, gofumpt carries frozen copies of `go/printer`, `go/format`, and `go/doc/comment` pinned to a specific Go release (Go 1.26.0 as of the latest version, which requires Go 1.25 or later to build)[^3]. This vendoring is what lets a pinned gofumpt version produce byte-identical output regardless of which Go toolchain runs it — the same reason it must be updated and re-released whenever upstream `gofmt` changes.

## Getting Started

```bash
go install mvdan.cc/gofumpt@latest
```

```bash
# Format in place, list files that changed
gofumpt -l -w .

# Enable the two extra opt-in rules (parameter grouping, no naked returns)
gofumpt -extra -l -w .

# Diff without writing (CI check)
gofumpt -d .
```

Most users never run the binary directly. gofumpt is wired into `gopls`, so enabling one setting makes format-on-save use it:

```json
// VS Code settings.json
"gopls": { "formatting.gofumpt": true }
```

## Architecture / How It Works

gofumpt is `gofmt` plus a post-parse AST pass. It parses Go source with `go/parser`, applies the standard `gofmt` normalization (using its vendored `go/printer`), and layers the added rules in the exported `format` package (`mvdan.cc/gofumpt/format`), which is importable so other tools can apply gofumpt formatting programmatically[^4].

The extra rules are pure AST transforms — they remove empty lines around lone block statements, force `std` imports into their own top group, rewrite `0755` to `0o755` on modules targeting Go 1.13+, collapse single-element `var (...)` groups into short assignments, add trailing commas that force multi-line call/composite-literal closings onto their own line, and normalize comment whitespace. `-s` (gofmt's simplification pass) is always on and hidden; the `-r` rewrite flag is removed in favor of running `gofmt -r` separately.

Two behaviors are worth internalizing because they surprise people:

- **`std` import grouping keys off import path shape, not a stdlib allowlist.** Any import path without a domain-like first segment (no dot) is treated as standard library and hoisted to the top group. Module paths that don't start with a domain (e.g. a bare `mycorp`) get grouped with the stdlib. The fix is to use a domain-qualified module path[^5].
- **Generated files and `vendor`/`testdata` directories are skipped** unless passed as explicit arguments, and `ignore` directives in `go.mod` are obeyed. This is intentional so the tool is safe to run recursively across a repo.

The `internal/govendor` package holds the frozen upstream copies. Files inherited from Go (`gofmt.go`, `internal.go`, `format/rewrite.go`, `format/simplify.go`) carry Google copyright headers and their own `LICENSE.google`; gofumpt's original code is separately BSD-3-Clause. Updating to a new Go release is a manual re-vendor, not an automated bump.

## Production Notes

**Pin the version.** gofumpt's output is intentionally allowed to change between minor releases as rules are added or refined. If two developers or CI run different gofumpt versions, they will fight over formatting. Pin it as a tool dependency (`go.mod` `tool` directive on Go 1.24+, or a `tools.go` blank import on older setups) and upgrade deliberately. A version bump in gofumpt frequently means a repo-wide reformatting diff.

**gopls is the recommended integration, not a shell-out.** Running gofumpt through `gopls` (`formatting.gofumpt: true`) is faster and stays consistent with the editor's other formatting, but the gofumpt version is then whatever `gopls` was built against — which can differ from your pinned CLI version and reintroduce the drift problem. Teams that care about exact output typically enforce the CLI version in CI regardless of what editors do locally.

**It does not manage imports.** gofumpt reorders and groups existing imports but, unlike `goimports`, never adds or removes them. Let `gopls` handle missing imports; if you must avoid `gopls`, chain the tools (`goimports file.go && gofumpt file.go`), accepting the per-file cold-start cost[^6].

**golangci-lint bundles it.** gofumpt is available both as a linter and as a formatter within golangci-lint. Running it as a linter reports non-gofumpt code as findings; running it as a formatter (via `golangci-lint fmt`) applies it. Mixing a standalone gofumpt with a golangci-lint-vendored gofumpt of a different version is a common source of "passes locally, fails in CI."

**Debugging formatting bugs.** Insert a `//gofumpt:diagnose` comment and run the tool; the comment is rewritten in place with the resolved version and language level, which is what maintainers ask for in bug reports.

## When to Use / When Not

**Use when:**
- Your team wants stricter, still-uncontroversial Go formatting with zero configuration and zero bikeshedding.
- You already run `gofmt` and want a superset that stays gofmt-compatible.
- You use `gopls` or golangci-lint and can flip one setting to adopt it.

**Avoid when:**
- You need a Go team-sanctioned, never-changing formatter — that is `gofmt` itself; gofumpt adds one maintainer's opinions and its output evolves across releases.
- You want configurable style (line length, alignment options) — gofumpt is deliberately non-configurable; look elsewhere.
- Your toolchain can't pin the formatter version, making cross-machine drift unavoidable.

## Alternatives

- golang/go `cmd/gofmt` — use instead when you want the canonical, unopinionated, forever-stable Go formatter with zero added rules.
- rinchsan/gosimports / incu6us/goimports-reviser — use when import management/grouping is the actual need rather than body formatting.
- mvdan/sh (`shfmt`) — same author's formatter, but for shell scripts, not Go.
- golangci/golangci-lint — use when you want gofumpt as one check inside a broader lint/format aggregator rather than a standalone binary.
- segmentio/golines — use alongside gofumpt when you specifically want automatic long-line wrapping, which gofumpt does not do.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-03-31 | Repository created as a personal stricter-gofmt fork[^1]. |
| v0.1.0 | 2021-01-05 | First tagged release; stabilized the added-rules set. |
| v0.2.0 | 2021-11-10 | `format` package exposed as an importable API. |
| v0.3.0 | 2022-02-22 | Rule refinements; Go 1.18 generics support era. |
| v0.4.0 | 2022-09-27 | Continued gofmt re-vendor and rule fixes. |
| v0.5.0 | 2023-04-09 | Maintenance and Go version tracking. |
| v0.6.0 | 2024-01-28 | Maintenance release. |
| v0.7.0 | 2024-08-16 | Maintenance release. |
| v0.8.0 | 2025-04-13 | Maintenance release. |
| v0.9.0 | 2025-09-02 | Re-vendor against a newer Go; minor rule updates. |
| v0.10.0 | 2026-05-04 | Fork updated to Go 1.26.0; requires Go 1.25+ to build[^3]. |

## References

[^1]: mvdan/gofumpt repository, created 2019-03-31. https://github.com/mvdan/gofumpt
[^2]: gofumpt README, "Why attempt to replace gofmt instead of building on top of it?" — the design never adds rules that disagree with gofmt. https://github.com/mvdan/gofumpt#frequently-asked-questions
[^3]: gofumpt README — "fork of gofmt as of Go 1.26.0, and requires Go 1.25 or later." https://github.com/mvdan/gofumpt/blob/master/README.md
[^4]: `mvdan.cc/gofumpt/format` package reference. https://pkg.go.dev/mvdan.cc/gofumpt/format
[^5]: gofumpt README — module import grouping and reserved path prefixes (golang/go#32819, #37641). https://github.com/mvdan/gofumpt#frequently-asked-questions
[^6]: gofumpt README — using gofumpt alongside goimports/gopls. https://github.com/mvdan/gofumpt#frequently-asked-questions

## Tags

go, golang, code-formatter, gofmt, gofumpt, static-analysis, developer-tooling, linter, code-style, cli
