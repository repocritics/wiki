# Masterminds/semver

> Parse, compare, sort, and constraint-check Semantic Versions in Go — the range engine most Go tooling reaches for.

[GitHub repo](https://github.com/Masterminds/semver) ·
[License: MIT](https://github.com/Masterminds/semver/blob/master/LICENSE.txt)

## Overview

`semver` is a small, single-purpose Go library for working with [Semantic Versions](https://semver.org): parsing them, sorting them, and — its most-used feature — checking whether a version satisfies a constraint range like `^1.2.3` or `>= 1.2 < 3.0.0 || >= 4.2.3`. It is maintained under the Masterminds organization, the same group behind the Sprig template functions and the Helm-adjacent Go toolchain, and it is a transitive dependency of a large slice of the Go ecosystem (Helm being the most visible consumer)[^1].

The library's defining tension is that Semantic Versioning defines *comparison* precisely (spec item 11) but says nothing about *ranges*. Constraint syntax — carets, tildes, hyphens, wildcards — was invented independently by npm, Cargo, Composer, and others, and they disagree. `semver` deliberately follows the npm/JavaScript and Cargo/Rust interpretation of `~` and `^` rather than, say, PHP/Composer's[^2]. This means the same constraint string can select different versions in `semver` than in a Composer or Python resolver, and that mismatch is the single most common source of surprise for people who assume "SemVer ranges" are universal.

The current line is v3, imported as `github.com/Masterminds/semver/v3`. v1 is unmaintained; v2 was an untagged branch built for the old `golang/dep` tool and has API-breaking differences[^3]. New code should always use v3.

## Getting Started

```bash
go get github.com/Masterminds/semver/v3
```

```go
package main

import (
	"fmt"

	"github.com/Masterminds/semver/v3"
)

func main() {
	// Parse. NewVersion coerces "v1.2" -> 1.2.0; StrictNewVersion would reject it.
	v, err := semver.NewVersion("v1.2.3-beta.1+build345")
	if err != nil {
		panic(err)
	}
	fmt.Println(v.Major(), v.Minor(), v.Patch(), v.Prerelease()) // 1 2 3 beta.1

	// Constraint check. Caret pins the major: ^1.2.3 == >=1.2.3 <2.0.0
	c, _ := semver.NewConstraint("^1.2.3")
	fmt.Println(c.Check(v)) // false — beta.1 is a prerelease, excluded by default
}
```

## Architecture / How It Works

The package is three loosely-coupled pieces: a `Version` value type, a `Collection` sort adapter, and a `Constraints` parser/evaluator.

`Version` stores the parsed major/minor/patch, prerelease, and metadata, plus the *original* string so a coerced input can be echoed back verbatim. Two entry points parse: `StrictNewVersion` accepts only spec-compliant SemVer 2.0.0; `NewVersion` coerces (leading `v`, missing minor/patch, leading zeros). Two package-level globals tune `NewVersion`: `CoerceNewVersion` (default true — enables CalVer-style inputs and leading zeros) and `DetailedNewVersionErrors` (slower, richer parse errors, only meaningful when coercion is off)[^4]. Because these are package globals, they are process-wide and not goroutine-configurable — a notable design constraint.

The critical semantic split is between **comparison** and **constraints**:

- `Compare`, `LessThan`, `GreaterThan`, etc. follow the spec exactly, *including* prereleases in ordering, so `1.2.3-beta.1 < 1.2.3`.
- `Constraints.Check()` follows range conventions borrowed from npm/Cargo, which by default treat a prerelease as *not* satisfying a range that doesn't itself mention a prerelease. `>=1.2.3` skips `1.4.0-alpha`; you opt prereleases back in with `>=1.2.3-0` or by setting `IncludePrerelease` on the `Constraints`.

Range sugar expands to bounded comparisons: `~1.2.3` → `>=1.2.3 <1.3.0`, `^1.2.3` → `>=1.2.3 <2.0.0` (with the special pre-1.0.0 rule where `^0.2.3` → `>=0.2.3 <0.3.0`, because before 1.0.0 the minor acts as the stability boundary). Wildcards (`x`, `X`, `*`) and hyphen ranges (`1.2 - 1.4.5`) desugar similarly. `Validate()` layers on top of `Check()` to return a slice of human-readable reasons a version failed.

Sorting is delegated to the standard library: `sort.Sort(semver.Collection(vs))` — `Collection` is just a `[]*Version` implementing `sort.Interface`.

## Production Notes

- **Range semantics are not portable.** If you are re-implementing another ecosystem's resolver (npm, PyPI, Composer, RubyGems), do not assume `semver`'s `^`/`~` match. They match npm/Cargo; they do not match Composer. Test against the source ecosystem's actual behavior before trusting cross-language equivalence[^2].
- **Prerelease exclusion is the top gotcha.** New users repeatedly report "my constraint doesn't match my RC build." That is by design: `>=1.2.3` excludes `1.3.0-rc.1`. The fixes — append `-0` to the range, or set `IncludePrerelease` — must be deliberate.
- **Whitespace is load-bearing.** `1.2 - 1.4.5` (spaces) is a hyphen range; `1.2-1.4.5` (no spaces) parses as a single version `1.2.0` with prerelease `1.4.5`. This is a genuine footgun when constraint strings are built by concatenation.
- **ASCII sort order for prereleases surprises people.** Per spec, `A`–`Z` sort before `a`–`z`, so `>=1.2.3-BETA` will match `1.2.3-alpha`. This is spec-correct, not a bug.
- **Global coercion flags.** `CoerceNewVersion` and `DetailedNewVersionErrors` are package-level, so a library that flips them affects every consumer in the process. Prefer `StrictNewVersion` when you want deterministic, local behavior.
- **Stability and cadence.** The project is marked "Active" and receives security tooling (CodeQL, gosec, daily fuzzing) and steady maintenance rather than rapid feature churn[^5]. The v3 API has stayed compatible for years, which is exactly what a widely-depended-on primitive should do.

## When to Use / When Not

**Use when:**
- You are on Go and need to parse, compare, or sort SemVer strings.
- You need npm/Cargo-style constraint ranges (`^`, `~`, hyphen, wildcard) evaluated in Go — the reason Helm and similar tools depend on it.
- You want a stable, small, MIT-licensed dependency with no transitive baggage.

**Avoid when:**
- You need range semantics that exactly mirror Composer, pip/packaging, or another non-npm resolver — you will fight the `^`/`~` differences.
- You only ever compare two exact versions with no ranges — `golang.org/x/mod/semver` is in the extended standard library and has zero third-party surface.
- Your "versions" are not SemVer at all (pure CalVer, Debian epochs, PEP 440) — coercion papers over some of this but the comparison rules remain SemVer's.

## Alternatives

- golang/mod (`golang.org/x/mod/semver`) — use when you want a Go-team-maintained, no-dependency comparator and don't need constraint ranges.
- blang/semver — use when you want an alternative Go SemVer type with its own range syntax and a long history in the ecosystem.
- hashicorp/go-version — use when you want version + constraint handling with HashiCorp's own (slightly different) range rules, common in Terraform-adjacent code.
- npm/node-semver — use when you are in JavaScript, or as the reference implementation whose `^`/`~` behavior `semver` intentionally mirrors.
- coreybutler/go-semver — smaller, less-used alternative; consider only if the above don't fit.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2015 | Original release under Masterminds; API later stabilized in v3[^3]. |
| 2.x | ~2016 | Untagged branch built for `golang/dep` by @sdboyer; API-breaking, never released as tags[^3]. |
| v3.0.0 | 2019 | Current stable line; module path `.../v3`, constraint-compatibility focus[^3]. |
| v3.4.0 | 2026-06 (approx.) | Continued v3 maintenance; last push 2026-06-24[^6]. |

## References

[^1]: Masterminds/semver README — "Work with Semantic Versions in Go." https://github.com/Masterminds/semver
[^2]: README, "Checking Version Constraints" — comparison follows npm/JS and Cargo/Rust `^`/`~` rather than PHP/Composer. https://github.com/Masterminds/semver#checking-version-constraints
[^3]: README, "Package Versions" — v1 unmaintained, v2 untagged `golang/dep` branch, v3 stable on master. https://github.com/Masterminds/semver#package-versions
[^4]: README, "Parsing Semantic Versions" — `CoerceNewVersion` and `DetailedNewVersionErrors` package variables. https://github.com/Masterminds/semver#parsing-semantic-versions
[^5]: README, "Security" — CodeQL, gosec, and daily fuzz testing. https://github.com/Masterminds/semver#security
[^6]: GitHub API repository metadata (fetched 2026-07-15): 1,425 stars, 164 forks, MIT, last push 2026-06-24. https://github.com/Masterminds/semver

## Tags

go, golang, semver, semantic-versioning, version-constraints, parsing, sorting, dependency-management, library, caret-tilde-ranges
