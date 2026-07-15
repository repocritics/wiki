# realm/SwiftLint

> A linter for Swift that enforces style and conventions, driven mostly by SwiftSyntax with a shrinking tail of SourceKit-based rules.

[GitHub repo](https://github.com/realm/SwiftLint) ·
[Official website](https://realm.github.io/SwiftLint) ·
[License: MIT](https://github.com/realm/SwiftLint/blob/main/LICENSE)

## Overview

SwiftLint is a static-analysis linter for Swift, originally built at Realm by JP
Simard and first released in 2015[^1]. Its rule set is loosely descended from the
now-archived GitHub Swift Style Guide and codifies conventions the Swift community
broadly agrees on — line length, force-unwrap avoidance, naming, whitespace, cyclomatic
complexity, and several hundred more. It ships ~200+ rules, split into *default* rules
(on unless disabled) and *opt-in* rules (off unless enabled), plus *analyzer* rules that
need full compiler information.

The project has quietly changed engines underneath. Early versions leaned heavily on
SourceKit and Clang to understand code; over several years most rules were rewritten
against SwiftSyntax[^2], the swift-syntax parser maintained by the Swift project. This is
faster and more precise for lexical/syntactic checks, but a handful of rules (those that
need type information) still call into SourceKit. That split defines SwiftLint's central
tradeoff: it is a mostly-syntactic linter that occasionally reaches for semantic data, and
the semantic path is slower and more fragile.

The second defining fact is governance. Realm was acquired by MongoDB in 2019, and the
company's investment in the tool tapered. SwiftLint is now largely community-maintained,
with much of the day-to-day work done by outside contributors[^3]. Notably, Swift Package
Manager plugin distribution moved to a separate community repository,
`SimplyDanny/SwiftLintPlugins`, which the README now recommends over consuming plugins from
`realm/SwiftLint` directly. The tool has never reached a 1.0 release — it has lived on the
0.x line for its entire history — which in practice signals ongoing churn rather than
instability.

## Getting Started

```bash
brew install swiftlint
```

Run it in a directory of Swift files (recurses by default):

```bash
swiftlint            # lint (default subcommand)
swiftlint --fix      # autocorrect fixable violations, then re-lint
```

Configuration lives in a `.swiftlint.yml` at the project root:

```yaml
disabled_rules:
  - trailing_whitespace
opt_in_rules:
  - empty_count
  - force_unwrapping
included:
  - Sources
excluded:
  - Sources/Generated
  - .build
line_length:
  warning: 120
  error: 200
identifier_name:
  min_length: 2
```

For CI, treat the exit code as pass/fail and add `--strict` to make warnings fail the
build.

## Architecture / How It Works

SwiftLint's pipeline per file is: read source → parse to a SwiftSyntax tree →
walk the tree with each enabled rule's visitor → collect violations → format via a
reporter. Rules are the unit of extension; each is a small type declaring its identifier,
severity, and a syntax visitor (or, for legacy rules, a SourceKit query).

Three rule classes matter operationally:

- **Syntactic rules** — the majority. Pure SwiftSyntax, no compiler needed. Fast, run
  under plain `swiftlint lint`.
- **Analyzer rules** — run under `swiftlint analyze` and require a compiler log
  (`--compiler-log-path` / `--compile-commands`) so they can see types. These catch things
  like unused declarations but need a full build first and are much slower.
- **Custom rules** — user-defined regex matchers configured directly in
  `.swiftlint.yml`. No Swift code required, but regex-based, so they see text, not syntax.

`--fix` (aka `--autocorrect`) only applies to rules that declare themselves correctable.
The README warns explicitly that SwiftLint expects *compilable* source — running `--fix` on
code that doesn't parse cleanly can produce confusing rewrites, which is why running the
linter as a pre-compile gate is discouraged.

Output is decoupled via reporters: `xcode` (the default, emits `warning:`/`error:` lines
Xcode surfaces inline), plus `json`, `csv`, `checkstyle`, `junit`, `html`, `markdown`,
`sonarqube`, `github-actions-logging`, `gitlab`, `emoji`, and `summary`. Toolchain
selection for the SourceKit path follows an override chain
(`XCODE_DEFAULT_TOOLCHAIN_OVERRIDE`, `TOOLCHAINS`, `xcrun -find swift`, then well-known
Xcode paths).

## Production Notes

**SwiftSyntax must match your Swift version.** Because parsing is done by a pinned
swift-syntax, source using newer language syntax than the SwiftLint build understands can
mis-parse or fail. Keep SwiftLint reasonably current with your toolchain; large version
gaps are a real source of spurious violations.

**Build-tool plugin adds cost to every build.** The SPM/Xcode build tool plugin runs
SwiftLint as a build phase on each target, on every build. On large projects this is
noticeable. Many teams instead run SwiftLint as a Run Script phase, or only in CI, to keep
incremental builds fast. The plugin also can't take arguments, so projects whose config
lives outside the package directory need a `parent_config` shim or the script approach.

**Xcode 15 sandboxing breaks the run-script path.** Xcode 15 flipped
`ENABLE_USER_SCRIPT_SANDBOXING` to `YES` by default, producing
`Sandbox: swiftlint(...) deny(1) file-read-data`. The fix is to set that build setting back
to `NO` for the linted target. Apple Silicon Homebrew installs to `/opt/homebrew/bin`,
which Xcode's build phase may not have on `PATH` — another common "not installed" false
alarm.

**Adopting on a legacy codebase.** A first run on an unlinted project can emit thousands of
violations. The `baseline` subcommand records current violations so only *new* ones fail,
letting teams ratchet quality without a big-bang cleanup.

**Plugin source matters.** Consume plugins from `SimplyDanny/SwiftLintPlugins`, not
`realm/SwiftLint` directly — the community repo keeps plugin code and releases in sync and
avoids overhead the README describes as "very troublesome" with the first-party plugins.

**Analyzer rules are a separate, heavier mode.** `swiftlint analyze` is not a superset of
`lint` you run casually; it needs a compiler log and a completed build, so it typically
lives in a dedicated CI job rather than the fast per-PR lint.

## When to Use / When Not

**Use when:**
- You want enforced, consistent Swift style across a team with a shared, versioned config.
- You want inline Xcode warnings and CI gating from the same tool.
- You need custom project-specific rules without writing a compiler plugin (regex custom
  rules).
- You're onboarding a large codebase incrementally (baseline).

**Avoid when:**
- You primarily want automatic *formatting* (whitespace/layout rewriting) — a dedicated
  formatter is a better fit than SwiftLint's narrower `--fix`.
- Per-build latency is critical and you can't move linting to CI.
- You need deep semantic/whole-program analysis — SwiftLint is mostly syntactic; analyzer
  rules cover only a slice.
- You want the assurance of a 1.0-stable tool with a rare-breaking-change contract.

## Alternatives

- nicklockwood/SwiftFormat — use when you want opinionated automatic code formatting rather
  than lint warnings; the two are commonly run together (format, then lint).
- apple/swift-format — use when you want the Apple-project-blessed formatter/linter built
  directly on swift-syntax with a single config model.
- peripheryapp/periphery — use when your goal is dead-code / unused-declaration detection;
  complementary to SwiftLint rather than a replacement.
- sleekbyte/Tailor — an older cross-platform Swift style checker; largely superseded and no
  longer actively developed.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2015 | Initial release; SourceKit/Clang-based rules[^1]. |
| 0.x | 2019 | `analyze` subcommand adds analyzer (type-aware) rules. |
| 0.x | 2020–2023 | Multi-year migration of rules from SourceKit to SwiftSyntax[^2]. |
| 0.54.0 | 2024 | `baseline` subcommand for incremental adoption. |
| 0.5x | 2025–2026 | Ongoing 0.x releases; SwiftSyntax-first, SPM plugins via SwiftLintPlugins[^4]. |

## References

[^1]: SwiftLint README — "A tool to enforce Swift style and conventions," loosely based on the archived GitHub Swift Style Guide. https://github.com/realm/SwiftLint
[^2]: SwiftLint README — rules "predominantly based on SwiftSyntax," with some still using Clang/SourceKit for type information. https://github.com/swiftlang/swift-syntax
[^3]: Realm was acquired by MongoDB (2019); SwiftLint is maintained largely by the community. https://github.com/realm/SwiftLint/graphs/contributors
[^4]: SwiftLintPlugins — community-maintained SPM plugin distribution recommended by the README. https://github.com/SimplyDanny/SwiftLintPlugins

## Tags

swift, linter, static-analysis, code-quality, swiftsyntax, sourcekit, ios, xcode, developer-tools, ci, style-guide
