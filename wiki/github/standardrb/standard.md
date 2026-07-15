# standardrb/standard

> A Ruby linter and formatter with a deliberately unconfigurable ruleset — RuboCop underneath, no bikeshedding on top.

[GitHub repo](https://github.com/standardrb/standard) ·
[License: MIT](https://github.com/standardrb/standard/blob/main/LICENSE.txt)

## Overview

Standard Ruby ports the ethos of StandardJS to Ruby: a linter and formatter that ships one fixed ruleset you either adopt or don't[^1]. It is a thin distribution layer over [RuboCop](https://github.com/rubocop/rubocop) — it does not implement its own cops, but instead locks a curated configuration of RuboCop's built-in rules plus `rubocop-performance`, and exposes it through a `standardrb` binary. Created and maintained by [Test Double](https://testdouble.com), it has been public since 2018[^2].

The defining decision is **unconfigurability**: you cannot change which rules run or how they are set. The argument is that the value of a linter comes from using one consistently, not from the particular choices, and that per-team rule negotiation is wasted effort. This is also its central tension — a team that disagrees with even one default has no supported project-wide override for that rule. The escape hatches are inline `# standard:disable` directives, file/glob `ignore:` lists, and additive plugins, but never re-tuning a cop that Standard already sets.

Because Standard is downstream of RuboCop, its behavior tracks RuboCop's. The maintainers ship on a roughly monthly cadence to absorb new and changed cops[^3], which means a version bump can surface new violations across an existing codebase even when your own code did not change. This is the price of the "we keep it current so you don't think about it" promise.

## Getting Started

```ruby
# Gemfile
gem "standard"
```

```bash
bundle install
bundle exec standardrb          # lint the current directory tree
bundle exec standardrb --fix    # apply fixes the tool is confident are safe
```

The binary is named `standardrb` to avoid collision with StandardJS's `standard`. A Rake task is also available:

```ruby
# Rakefile
require "standard/rake"
```

```bash
rake standard          # lint
rake standard:fix      # autocorrect
```

Standard exits 0 when clean and reports violations otherwise, making it directly usable in CI with no plugin required.

## Architecture / How It Works

Standard is a configuration and runner shim, not a rules engine. Its moving parts:

- **Locked config.** The ruleset lives in `config/base.yml` (plus the `standard-custom` and `standard-performance` companion gems, which are default plugins). At runtime Standard merges these into a RuboCop configuration and invokes RuboCop's own runner. CLI arguments Standard doesn't recognize are forwarded to RuboCop.
- **lint_roller plugin protocol.** Extensions are gems implementing [lint_roller](https://github.com/standardrb/lint_roller) — `standard-rails`, `standard-sorbet`, `standard-performance`, and `standard-custom` all use it. Plugins can *add* cops but cannot reconfigure ones Standard already sets. Merge semantics are **first-in-wins**: once any plugin or extension configures a cop, later ones cannot override it, and plugins are applied before `extend_config` files.
- **Comment directives.** `# standard:disable Cop/Name`, `# standard:enable Cop/Name`, and `# standard:disable all` are Standard's own directive syntax (distinct from RuboCop's `# rubocop:disable`), handled during the run.
- **`ruby_version` targeting.** Standard sets RuboCop's `TargetRubyVersion`, inferring it from `.ruby-version`, `.tool-versions`, `Gemfile.lock`, and `*.gemspec`. Wrong inference means cops target the wrong Ruby and emit false positives — hence the explicit `ruby_version:` YAML key for gems that support old rubies.
- **Two language servers.** Both ship in the gem: a [Ruby LSP](https://github.com/Shopify/ruby-lsp) add-on (auto-loaded when `standard` is in the Gemfile; pull diagnostics + code actions) and a standalone server via `standardrb --lsp` (push diagnostics + formatting, no code actions). The maintainers recommend the Ruby LSP add-on where possible.

Running Standard's ruleset through the plain `rubocop` CLI is possible but **explicitly unsupported** and may break on any release[^4].

## Production Notes

**Version bumps move the goalposts.** Because Standard tracks RuboCop monthly, upgrading the gem can introduce new violations in unchanged code. Pin the version in your Gemfile and treat Standard upgrades as their own reviewed change. For adopting Standard on a large legacy codebase, `standardrb --generate-todo` writes a `.standard_todo.yml` that suppresses every existing violation so you can fix incrementally rather than in one commit.

**No project-wide rule override — by design.** If a single default cop is noisy for your project, there is no supported config knob to disable it globally. Your options are inline directives, `ignore:` globs (optionally scoped to specific cops per glob), or accepting it. Teams that need to silence a rule across the whole repo will find this genuinely constraining; that friction is the product's thesis, not a bug.

**Performance on large repos.** Standard runs the full curated ruleset, which is heavier than a hand-pruned RuboCop config. RuboCop is single-threaded by default; set `parallel: true` in `.standard.yml` (or pass through) on big codebases. There is no incremental cache beyond RuboCop's own.

**Autocorrect tiers.** `--fix` applies only corrections the tool considers behavior-preserving; `--fix-unsafely` adds corrections that may change runtime behavior. Many teams run `--fix` continuously and reserve `--fix-unsafely` for reviewed passes with tests and version control as the safety net.

**Plugin conflicts are silent.** With first-in-wins merging, two plugins that both configure the same cop resolve to whichever loaded first, with no error. Keep the plugin list short and ordered intentionally.

**Editor integration reality.** The `standardrb --lsp` internal server lacks code actions ("Quick Fixes"), so editors driven by it get diagnostics and formatting but not one-click fixes; the Ruby LSP add-on path is the fuller experience.

## When to Use / When Not

**Use when:**
- You want a linter/formatter decision made *for* you and want to stop arguing about style.
- You value consistency across many repos or a large team over per-project tuning.
- You want RuboCop's engine and cop coverage without maintaining a `.rubocop.yml`.
- You want automatic formatting plus lint in one tool, with LSP and CI support included.

**Avoid when:**
- Your team has strong, specific style opinions it needs to enforce — use RuboCop directly.
- You must disable or retune particular rules project-wide (Standard offers no supported path).
- You want formatting only, with no linting layer.
- You need to freeze the ruleset for years; Standard's monthly tracking means the ruleset evolves.

## Alternatives

- rubocop/rubocop — the fully configurable engine Standard wraps; use it directly when you need control over every cop.
- ruby-syntax-tree/syntax_tree — AST-based formatter that reformats from scratch and ignores prior style; use when you want pure formatting decoupled from lint rules.
- ruby-formatter/rufo — an opinionated formatter (no linting); use when you want auto-format only and no cop violations.
- Shopify/rubocop-shopify — a shareable RuboCop config you *can* override; use when you want strong defaults but reserve the right to change rules.
- standard/standard — the original StandardJS for JavaScript; use for JS/TS projects, not Ruby.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-11 | Public release; StandardJS ethos ported to Ruby, built on RuboCop[^2]. |
| 1.0 | 2021-05 | First stable major (approx.); unconfigurable-config model established[^5]. |
| lint_roller era | 2023 | `lint_roller` plugin protocol + `standard-rails` / `standard-sorbet`; LSP support added. |
| v1.55.0 | 2026-07 | Current release line; monthly RuboCop tracking, actively maintained[^3]. |

## References

[^1]: Standard Ruby README — "Wait, did you say unconfigurable configuration?". https://github.com/standardrb/standard#wait-did-you-say-unconfigurable-configuration
[^2]: `standardrb/standard` repository, created 2018-11-07. https://github.com/standardrb/standard
[^3]: Standard Ruby README on monthly release cadence tracking new RuboCop rules. https://github.com/standardrb/standard#readme
[^4]: Standard Ruby README — "Running Standard's rules via RuboCop" (explicitly unsupported). https://github.com/standardrb/standard#running-standards-rules-via-rubocop
[^5]: Standard Ruby releases page (version-date verification). https://github.com/standardrb/standard/releases

## Tags

ruby, linter, formatter, rubocop, static-analysis, code-style, autocorrect, lsp, developer-tooling, ci
