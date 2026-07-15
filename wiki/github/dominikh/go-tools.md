# dominikh/go-tools

> Staticcheck — a static-analysis toolchain for Go that finds bugs, flags performance issues, and enforces simplifications and style.

[GitHub repo](https://github.com/dominikh/go-tools) ·
[Official website](https://staticcheck.dev) ·
[License: MIT](https://github.com/dominikh/go-tools/blob/master/LICENSE)

## Overview

`dominikh/go-tools` is the repository behind **Staticcheck**, the most widely used third-party linter for Go, written and maintained primarily by Dominik Honnef under the `honnef.co/go/tools` import path[^1]. It began as a collection of separate tools (`staticcheck`, `gosimple`, `unused`, once bundled as `megacheck`) and consolidated into a single `staticcheck` binary whose checks are grouped by prefix: `SA` for correctness bugs, `S` for simplifications, `ST` for style, `U` for unused code, and `QF` for quickfixes[^2].

The defining tension is scope versus stability. Staticcheck aims to catch real bugs (misused `sync` primitives, impossible type assertions, incorrect `Printf` verbs, unreachable code) with a low false-positive rate, which makes it the linter teams trust to run in CI and fail builds. But it is coupled to the Go toolchain in a way that most linters are not: it performs full type-checking of your program, so it must be built with a Go version at least as new as the code it analyzes, and new language features (notably generics) require an updated Staticcheck release before they can be analyzed at all[^3].

The repository also ships smaller utilities (`structlayout`, `structlayout-optimize`, `structlayout-pretty`) for inspecting and repacking struct field padding. The supporting libraries are deliberately unstable: the README warns that only the command-line tools have a supported interface, and library consumers should expect backwards-incompatible changes.

## Getting Started

```bash
# Install a pinned release (recommended over @latest)
go install honnef.co/go/tools/cmd/staticcheck@2025.1

# Run against the current module
staticcheck ./...
```

```go
// Staticcheck flags this: the loop variable is captured by reference
// (pre-Go 1.22 semantics) and Printf verb mismatch.
package main

import "fmt"

func main() {
	xs := []int{1, 2, 3}
	fmt.Printf("%d\n", "not a number") // SA5009: %d expects int, got string
	for _, x := range xs {
		defer func() { fmt.Println(x) }() // SA... loop capture warning (older Go)
	}
}
```

Configuration lives in a `staticcheck.conf` file at the module root (HCL-like syntax), and individual findings can be suppressed inline with `//lint:ignore Check reason` directives.

## Architecture / How It Works

Every Staticcheck check is implemented as an analyzer conforming to the `golang.org/x/tools/go/analysis` `Analyzer` interface[^4]. This is the same framework `go vet` uses, and it is the key architectural decision: because the checks are standard analyzers, they run unmodified inside multiple *drivers*.

- **The `staticcheck` CLI** is one driver. It loads packages with `go/packages`, type-checks them, builds SSA (single static assignment) form for the deeper flow-sensitive checks, and runs the analyzer set.
- **gopls** (the official Go language server) embeds a subset of Staticcheck's analyzers, so editor diagnostics can come from Staticcheck without invoking the CLI[^5].
- **golangci-lint** bundles Staticcheck as one of its linters, running it in-process alongside dozens of others and sharing the loaded/type-checked program to avoid redundant work[^6].

Checks fall into families by cost. The `SA` bug checks and `S`/`ST` checks are mostly package-local. The `U1000` unused check is different: identifying dead code correctly requires whole-program reachability, so it loads and considers the entire package graph rather than one package at a time. This is why "unused" is both the most valuable and the most resource-intensive check.

Releases use calendar versioning (`2022.1`, `2023.1`, `2025.1`, …) rather than semver. Each release documents which Go versions it can be built with and which it can analyze; the general rule is that Staticcheck tracks the current Go release and drops support for old ones over time.

## Production Notes

- **Build-version coupling is the top footgun.** Staticcheck must be compiled with a Go toolchain at least as new as the code under analysis. A CI image pinned to an older Go, or a stale `@latest` binary cached in `$GOBIN`, will fail or silently mis-analyze code that uses newer syntax. Pin the release tag (`@2025.1`) and rebuild it when you bump Go.
- **Generics required a full release.** Code using type parameters could not be analyzed until Staticcheck gained generics support (the 2023.1 line)[^3]. Teams that adopted generics early hit hard failures until they upgraded.
- **Whole-program `U1000` is memory-heavy.** On large monorepos the unused check dominates runtime and RAM. If Staticcheck OOMs or crawls in CI, disabling `U1000` (or scoping runs per-module) is the usual fix.
- **Non-compiling code is not analyzed.** Because it type-checks first, Staticcheck reports load/type errors and skips analysis when the program does not build. It is not a "lint the broken file" tool.
- **False-positive policy is conservative but not zero.** The project treats noisy checks as bugs and tunes for signal, but `SA` checks around `sync`, contexts, and deferred calls occasionally flag intentional patterns; use `//lint:ignore` with a reason rather than disabling a whole check globally.
- **Caching.** Staticcheck caches results (keyed on inputs) so repeat runs are fast; a cold CI cache pays full cost on every run unless you persist the cache directory.
- **Run it through one driver, not three.** Running the CLI, gopls, and golangci-lint's Staticcheck simultaneously duplicates work and can surface version-skew disagreements. Pick the CLI for CI and let editors use gopls.

## When to Use / When Not

**Use when:**
- You want a high-signal bug finder in CI that can gate merges on real defects, not just style.
- You need dead-code detection (`U1000`) beyond what `go vet` offers.
- You want checks that also work in your editor (via gopls) and in an aggregator (via golangci-lint) from one analyzer source.

**Avoid / reconsider when:**
- You cannot keep the linter's Go build version current — the coupling will bite you.
- You want dozens of linters with one config: use golangci-lint (which includes Staticcheck) rather than the standalone CLI.
- You need security-specific rules (taint/injection): pair with a dedicated tool; Staticcheck is not a SAST product.
- Your codebase does not reliably compile in CI — analysis will be skipped.

## Alternatives

- golangci-lint/golangci-lint — an aggregator that already bundles Staticcheck; use it when you want many linters behind one config and runner.
- golang/go (`go vet`) — the official, always-available checker; use it when you want zero extra dependencies and a smaller, conservative check set.
- securego/gosec — use it when your priority is security/SAST rules rather than correctness and style.
- mgechev/revive — use it when you want a fast, highly configurable `golint` successor focused on style/lint rules.
- kisielk/errcheck — use it when you specifically need exhaustive unchecked-error detection.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017 | Repository created; early `staticcheck`/`gosimple`/`unused` tools, bundled as `megacheck`[^1]. |
| 2020.1 | 2020 | Consolidation into a single `staticcheck` binary and unified check prefixes[^2]. |
| 2022.1 | 2022 | Calendar-versioned release line; check set expanded, `QF` quickfixes present. |
| 2023.1 | 2023 | Support for analyzing generics (type parameters)[^3]. |
| 2025.1 | 2025 | Current release line tracking recent Go versions. |

## References

[^1]: Staticcheck README and project site. https://staticcheck.dev/ and https://github.com/dominikh/go-tools
[^2]: Staticcheck checks documentation (SA/S/ST/U/QF prefixes). https://staticcheck.dev/docs/checks/
[^3]: Staticcheck release notes / configuration and Go-version support. https://staticcheck.dev/docs/running-staticcheck/cli/
[^4]: `go/analysis` framework, golang.org/x/tools. https://pkg.go.dev/golang.org/x/tools/go/analysis
[^5]: gopls Staticcheck integration. https://github.com/golang/tools/blob/master/gopls/doc/analyzers.md
[^6]: golangci-lint linters (includes Staticcheck). https://golangci-lint.run/usage/linters/

## Tags

go, static-analysis, linter, code-quality, staticcheck, go-analysis, developer-tools, cli, dead-code-detection, ci
