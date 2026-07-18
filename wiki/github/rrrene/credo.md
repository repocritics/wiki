# rrrene/credo

> Static code analysis for Elixir that treats the linter as a teaching tool — consistency and design feedback, not just rule enforcement.

[GitHub repo](https://github.com/rrrene/credo) ·
[Official website](http://credo-ci.org/) ·
[License: MIT](https://github.com/rrrene/credo/blob/master/LICENSE)

## Overview

Credo is the de facto standard linter for Elixir, created and still primarily maintained by René Föhring since 2015[^1]. It analyzes source files without compiling them and reports issues in five categories: consistency, readability, refactoring opportunities, software design, and warnings. Its stated focus — "code consistency and teaching" — is visible in the output: `mix credo explain` prints a rationale for every issue, which is why many Elixir teams use it as an onboarding tool as much as a CI gate.

The defining design choice is that Credo works on the *quoted form* (Elixir's AST via `Code.string_to_quoted`) rather than on compiled BEAM artifacts. That makes it fast, dependency-light (`runtime: false`), and able to run on code that doesn't compile — but it also means Credo has no type information and cannot expand macros. It is a style-and-structure critic, not a correctness checker; type-level analysis in Elixir belongs to Dialyzer, and security scanning to Sobelow. The healthy setup treats these as three orthogonal tools.

At ~5.2k stars and 449 forks it is one of the most-adopted packages in the Elixir ecosystem, and it remains actively maintained (last push June 2026), though the bus factor is real: it is substantially a single-author project.

## Getting Started

Add Credo to `mix.exs` as a dev/test-only dependency:

```elixir
defp deps do
  [
    {:credo, "~> 1.7", only: [:dev, :test], runtime: false}
  ]
end
```

```bash
mix deps.get
mix credo             # standard run (higher-priority issues only)
mix credo --strict    # include low-priority issues
mix credo explain lib/my_app/worker.ex:15:7   # why is this an issue?
mix credo gen.config  # writes .credo.exs for customization
```

A minimal `.credo.exs` tweak — disabling one check and tightening another:

```elixir
%{
  configs: [
    %{
      name: "default",
      checks: %{
        disabled: [{Credo.Check.Design.TagTODO, []}],
        extra: [{Credo.Check.Refactor.CyclomaticComplexity, max_complexity: 8}]
      }
    }
  ]
}
```

## Architecture / How It Works

A run is organized as a `Credo.Execution` pipeline: parse CLI args, resolve config, load source files, run checks, format output. Each source file becomes a `Credo.SourceFile` struct holding the raw source plus its AST; checks are modules implementing the `Credo.Check` behaviour with a category, a base priority, per-check params, and a `run` function that traverses the AST and emits issues[^1].

Two execution modes coexist:

- **Per-file checks** (the majority) run in parallel across files.
- **Consistency checks** are cross-file and two-phase: first collect each file's convention (e.g. which operator spacing or naming scheme it uses), then flag the minority against the codebase-wide majority. This is Credo's most novel idea — the "correct" style is whatever your codebase already mostly does.

Extensibility has three tiers: inline config comments (`# credo:disable-for-next-line`), custom checks (any module implementing the behaviour, registered in `.credo.exs`), and plugins — packages that hook the execution pipeline to add checks, CLI commands, or output formats[^2]. Output formatters include the default report, `--format=json`, flycheck-style one-liners, and SARIF for GitHub code scanning integration[^2].

Because analysis happens pre-expansion, macro-heavy code is Credo's blind spot: DSLs (Phoenix routers, Ecto schemas, Absinthe types) are seen as literal macro calls, which occasionally produces false positives and means DSL-internal problems go unseen.

## Production Notes

- **Exit status is a bitmask, not a boolean.** Issue categories map to bits and the exit code is their union, so CI can distinguish "warnings found" from "consistency drift". Use `--mute-exit-status` for advisory runs; most teams instead gate CI on the default behavior.
- **`--strict` surprises.** The default run hides low-priority issues. Teams routinely discover a backlog of hundreds of suppressed issues the first time someone runs `--strict`. Decide early which mode CI enforces.
- **Legacy codebases: use `mix credo diff`.** `diff --from-git-merge-base main` reports only issues introduced by the current branch, which is the practical adoption path for large existing apps — enforcing a clean full run on a mature codebase up front usually stalls adoption.
- **Consistency checks can flip.** Since they enforce the majority convention, a large refactor that shifts the 60/40 balance makes the *previously passing* side start failing. This is by design but regularly confuses CI archaeology.
- **Minor upgrades can add or tune checks**, changing what a previously green CI reports. Pin the version and read the changelog before bumping[^2].
- **Overlap with `mix format`.** Since Elixir 1.6 shipped a canonical formatter[^3], Credo's whitespace/layout checks matter less; its lasting value is in the design, refactoring, naming, and warning categories that a formatter cannot express.
- **It is not a security or type tool.** Pair with Sobelow for Phoenix security checks[^4] and Dialyzer (via dialyxir) for success-typing analysis. Credo passing says nothing about either.

## When to Use / When Not

**Use when:**
- You maintain an Elixir codebase with more than one contributor and want automated, explainable style and design feedback in CI.
- You are onboarding developers new to Elixir — `explain` output doubles as idiom documentation.
- You want codebase-relative consistency enforcement rather than a fixed global style dogma.

**Avoid when:**
- You only care about formatting — `mix format --check-formatted` is built in and sufficient.
- You expect type or correctness guarantees — that is Dialyzer's job, not a lint pass over unexpanded AST.
- Your codebase is dominated by macro DSLs; expect limited signal and some false positives there.

## Alternatives

- jeremyjh/dialyxir — use instead (or alongside) when you want type-level discrepancy analysis via Dialyzer's success typing rather than style checks.
- nccgroup/sobelow — use when the goal is security static analysis for Phoenix applications.
- adobe/elixir-styler — use when you would rather auto-rewrite style issues as a formatter plugin than have a linter report them.
- elixir-lang/elixir (`mix format`) — use alone when layout consistency is your only requirement.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015-09 | Initial development; repo created 2015-09-27. |
| 1.0 | 2018-12 | Stable release; config format consolidated[^2]. |
| 1.3 | 2020 | Plugin system for third-party checks, commands, formats[^2]. |
| 1.4 | 2020 | `mix credo diff` — analyze only changes vs. a git ref[^2]. |
| 1.6 | 2021 | SARIF export for code-scanning integration[^2]. |
| 1.7 | 2023 | Current release series; `~> 1.7` is the recommended dep[^1]. |

## References

[^1]: Credo documentation on Hexdocs. https://hexdocs.pm/credo/
[^2]: Credo CHANGELOG. https://github.com/rrrene/credo/blob/master/CHANGELOG.md
[^3]: Elixir v1.6 release announcement (introduces `mix format`) — 2018-01-17. https://elixir-lang.org/blog/2018/01/17/elixir-v1-6-0-released/
[^4]: Sobelow — security-focused static analysis for Phoenix. https://github.com/nccgroup/sobelow

## Tags

elixir, static-analysis, linter, code-quality, code-consistency, mix-task, ast-analysis, developer-tools, ci-tooling
