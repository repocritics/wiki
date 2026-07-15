# rubocop/rubocop

> A Ruby static analyzer and formatter that enforces the community Ruby Style Guide — and argues with you about it.

[GitHub repo](https://github.com/rubocop/rubocop) ·
[Official website](https://docs.rubocop.org) ·
[License: MIT](https://github.com/rubocop/rubocop/blob/master/LICENSE.txt)

## Overview

RuboCop is a Ruby linter and code formatter, started by Bozhidar Batsov in 2012[^1] as a way to mechanically enforce the community-driven [Ruby Style Guide](https://rubystyle.guide). It occupies a single position in the Ruby ecosystem that few languages consolidate this way: it is simultaneously the de facto style linter, the correctness linter, the complexity-metrics tool, and the autoformatter. Where a JavaScript project might run ESLint plus Prettier, and a Python project Ruff plus Black, a Ruby project usually runs RuboCop for all of it.

The defining tension is opinionatedness. Out of the box RuboCop enforces hundreds of rules ("cops"), many of them purely aesthetic (single vs. double quotes, `Hash#each` vs. `Hash#map`, method length). Historically this made a fresh `rubocop` run on an existing codebase emit thousands of offenses, and the defaults themselves — Batsov's and the core team's opinions — are contentious enough that an entire counter-project (Standard) exists specifically to eliminate the configuration debate. RuboCop's answer is that essentially every cop is configurable or disableable via `.rubocop.yml`; the cost is that teams spend real time curating that file.

Since version 1.0 (2020) the project has been deliberately stable: API and cop behavior are frozen within a major version, new cops ship "pending" (opt-in) rather than enabled, and breaking changes are reserved for major releases[^2]. This stability commitment is what makes it viable as a CI gate for large, long-lived Rails codebases, which are its dominant real-world use case.

## Getting Started

```sh
gem install rubocop
```

Or, more commonly, pin it in the `Gemfile` (with `require: false`, since it is a standalone tool):

```rb
gem 'rubocop', '~> 1.82', require: false
```

Run it in a project root:

```sh
rubocop                 # lint the whole project
rubocop -a              # autocorrect safe offenses
rubocop -A              # autocorrect all offenses, including unsafe ones
rubocop app/models/user.rb   # a single file
```

A minimal `.rubocop.yml` that adopts new cops and relaxes a couple of defaults:

```yaml
AllCops:
  TargetRubyVersion: 3.3
  NewCops: enable          # opt into "pending" cops automatically
Metrics/MethodLength:
  Max: 20
Style/Documentation:
  Enabled: false
```

## Architecture / How It Works

RuboCop parses Ruby into an AST and walks it, running each enabled cop against relevant nodes.

- **Parsing.** Historically RuboCop parsed with the `parser` gem (whitequark/parser), a pure-Ruby parser that produces a whitequark-format AST and can target multiple Ruby syntax versions independently of the running interpreter. RuboCop later added support for **Prism**, the parser that became Ruby's default in 3.4[^3], as an alternative front end. The `rubocop-ast` gem — extracted from the core around 1.0 — wraps the raw AST with a richer node API and the node-pattern matching DSL cops use.
- **Cops and departments.** Each cop is a class that subscribes to node types (e.g. `on_send`, `on_def`). Cops are grouped into **departments**: `Layout` (whitespace/formatting), `Style` (idiom preferences), `Lint` (probable bugs), `Metrics` (size/complexity), `Naming`, `Security`, `Bundler`, `Gemspec`, and `Migration`. Layout cops are the ones that make RuboCop a formatter; Lint cops are the ones that make it a bug finder.
- **Autocorrection.** Cops can implement a corrector that rewrites source via `rubocop-ast`'s source-rewriting layer. Corrections are tagged **safe** or **unsafe**: safe corrections cannot change runtime behavior (whitespace, quote style); unsafe ones might (e.g. replacing a method that has subtly different semantics), and only run under `-A`. This safe/unsafe split is a core design distinction and a frequent source of "why didn't `-a` fix this."
- **Configuration resolution.** `.rubocop.yml` files inherit — from `config/default.yml` at the bottom, up through `inherit_from`/`inherit_gem`, and down the directory tree. The effective config for any file is a merge, which is powerful and also the reason config debugging (`rubocop --show-cops`, `--debug`) is sometimes necessary.
- **Extensions.** Cop sets for specific ecosystems live in separate gems — `rubocop-rails`, `rubocop-rspec`, `rubocop-performance`, `rubocop-minitest`, `rubocop-thread_safety` — loaded via `require` in `.rubocop.yml`. The core gem intentionally stays framework-agnostic.

## Production Notes

**Startup latency dominates small runs.** Booting Ruby, loading RuboCop, and parsing config takes on the order of a second before any file is analyzed — painful for editor-on-save or pre-commit hooks on a handful of files. The fix is **server mode** (`rubocop --server` / `--start-server`), a background process introduced in 2022 that keeps RuboCop resident and avoids repeated boot cost[^4]. There is also a built-in **LSP server** (`rubocop --lsp`) so editors get diagnostics and formatting without shelling out per keystroke[^5]. Both are opt-in; a naive `rubocop` in a git hook will not use them.

**Adopting on an existing codebase.** Running RuboCop cold on a large legacy app produces a wall of offenses. The intended workflow is `rubocop --auto-gen-config`, which writes a `.rubocop_todo.yml` cataloguing every current violation so they are grandfathered out, letting you enforce the rules going forward and burn down the TODO file incrementally. Skipping this step is the most common reason teams bounce off the tool.

**Upgrades add cops.** Minor releases regularly add new cops. If `NewCops: enable` is set, an upgrade can surface new offenses and fail CI on a build that touched nothing relevant; if it is not set, new cops sit dormant and you slowly drift from the project's intended defaults. Neither choice is free — teams pick a policy and pin the version (`~> 1.x`).

**Performance on big trees.** Full-project runs on large monorepos take real wall-clock time. Mitigations: the result `--cache` (on by default) skips unchanged files, `--parallel` uses multiple cores, and CI typically runs `rubocop` only on changed files for PR feedback while doing a full run nightly.

**Autocorrect is not a formatter guarantee.** Unlike Prettier/gofmt, RuboCop does not promise a single canonical output — two configs produce two different "correct" formattings, and `-A` can occasionally produce code that itself triggers a different cop, requiring a second pass. Treat `-A` output as a draft to review, not a black box.

## When to Use / When Not

**Use when:**
- You have a Ruby or Rails codebase and want one tool for style, lint, complexity, and formatting.
- You want a CI gate that enforces consistent style across a team without per-PR bikeshedding.
- You want gradual adoption on legacy code via `.rubocop_todo.yml`.
- You rely on ecosystem cop packs (Rails, RSpec, performance) that only exist for RuboCop.

**Avoid when:**
- You want zero configuration and no style debates — Standard (a RuboCop wrapper with locked rules) removes the config surface entirely.
- You need type checking or dead-code/flow analysis — that is Sorbet/Steep territory, not RuboCop.
- You want a formatter with a single guaranteed canonical output like gofmt — RuboCop's formatting is config-dependent.
- Startup cost matters and you cannot run the server/LSP (e.g. constrained CI shells invoking it per file).

## Alternatives

- testdouble/standard — RuboCop under the hood with a fixed, unconfigurable ruleset; use it when you want to end the config argument entirely.
- sorbet/sorbet — gradual static type checker for Ruby; use it when you need type safety, not style, and pair it with RuboCop rather than replacing it.
- ruby/prism — the parser (now Ruby's default) that RuboCop can build on; relevant if you are writing your own analysis tooling instead of consuming cops.
- fables-tales/rubyfmt or ruby-syntax-tree/syntax_tree — pure formatters; use when you only want deterministic formatting and no linting opinions.
- troessner/reek — code-smell detector focused on design/complexity; complements RuboCop rather than replacing it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9 | 2013-07 | Early public release; rapid churn in defaults and cops[^1]. |
| 0.50 | 2017-09 | Departments reorganized; formatting cops consolidated. |
| 0.72 | 2019 | `Cop` autocorrection API modernized; extension split underway. |
| 0.90 | 2020-08 | `rubocop-ast` extracted into its own gem. |
| 1.0 | 2020-10 | Stability guarantees begin; pending-cops model, frozen within major[^2]. |
| 1.31 | 2022 | Server mode for fast repeated invocation[^4]. |
| 1.53 | 2023 | Built-in LSP server for editor integration[^5]. |
| 1.x | 2024–2026 | Ongoing; Prism parser support added alongside the `parser` gem[^3]. |

## References

[^1]: Bozhidar Batsov and contributors, RuboCop project history and README. https://github.com/rubocop/rubocop
[^2]: RuboCop versioning policy — stability within major versions, pending cops. https://docs.rubocop.org/rubocop/versioning.html
[^3]: Prism, Ruby's parser (default since Ruby 3.4); RuboCop supports it as an alternative front end. https://github.com/ruby/prism
[^4]: RuboCop server mode documentation. https://docs.rubocop.org/rubocop/usage/server.html
[^5]: RuboCop built-in LSP server. https://docs.rubocop.org/rubocop/usage/lsp.html

## Tags

ruby, linter, static-analysis, code-formatter, code-quality, ast, cli, rails, ruby-style-guide, developer-tools
