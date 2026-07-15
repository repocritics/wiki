# simplecov-ruby/simplecov

> Code coverage for Ruby — a filtering, grouping, and merging layer on top of the standard library's `Coverage` module.

[GitHub repo](https://github.com/simplecov-ruby/simplecov) ·
[License: MIT](https://github.com/simplecov-ruby/simplecov/blob/main/LICENSE)

## Overview

SimpleCov is the de facto code coverage tool for Ruby. It does not instrument code itself — it delegates the actual line/branch/method counting to Ruby's built-in `Coverage` standard library[^1] and adds everything around it: filtering out files you don't care about, grouping files into buckets, merging results across multiple test suites, applying minimum-coverage thresholds, and rendering an HTML (or JSON) report. Christoph Olszowka started the project in 2010, when Ruby 1.9 first shipped a native `Coverage` module; before that, `rcov` was the common tool and it stopped working on 1.9. The repo now lives under the `simplecov-ruby` organization.

The defining design choice is that SimpleCov is framework-agnostic and process-local. It hooks whatever process it is required into and reports on whatever code that process executes — so the same two lines work under Minitest, RSpec, Cucumber, or a running Rails server. The flip side is the single most common footgun: `SimpleCov.start` must run **before** your application code is loaded, or the `Coverage` library never sees those files and reports them as untracked. This bites hard with anything that keeps the app resident between runs (Spring, Zeitwerk eager-loading, preforking test runners).

The other structural fact worth internalizing is result merging. A real Ruby project runs coverage across several independent processes (unit + Cucumber + system specs, or N parallel CI shards). SimpleCov's job is to reconcile those into one honest number, and its merging model — and how that model changed in 0.18 — is where most operational pain lives.

## Getting Started

```ruby
# Gemfile
gem 'simplecov', require: false, group: :test
```

```ruby
# spec/spec_helper.rb (or test/test_helper.rb) — the FIRST lines in the file
require 'simplecov'
SimpleCov.start 'rails' do
  add_filter '/spec/'
  add_group 'Services', 'app/services'
  minimum_coverage line: 90
end

# ... your existing requires start below this ...
```

Run the suite, then open `coverage/index.html`. The `'rails'` argument is a bundled profile that pre-groups Controllers, Models, Helpers, and Libraries. The default HTML formatter and a JSON formatter are built in (they were once the separate `simplecov-html` and `simplecov_json_formatter` gems) and need no extra dependencies.

To gate CI and see per-branch results, opt into branch coverage:

```ruby
SimpleCov.start do
  enable_coverage :branch      # count both arms of each conditional
  minimum_coverage line: 90, branch: 80
end
```

A common convention is to guard SimpleCov behind an env flag (`SimpleCov.start if ENV['COVERAGE']`) so local runs skip the report while CI always produces one.

## Architecture / How It Works

The pipeline is: enable `Coverage` → run the suite → at-exit hook collects the raw result → filter → group → merge → format.

- **Collection.** `SimpleCov.start` calls `Coverage.start` with the enabled criteria and installs an `at_exit` hook. All counting is C-level in Ruby's stdlib; SimpleCov never parses your source to count lines. This makes overhead low — a couple of seconds on a 10-minute suite — which is why SimpleCov runs coverage on every run rather than on demand.
- **Criteria.** Line coverage is always on. Branch, method, oneshot-line, and eval coverage are opt-in (`enable_coverage :branch`). Branch coverage answers "did both arms of this conditional run" — line coverage cannot — but it is not strictly superior: a file with no conditionals reports 100% branch (0-of-0) while hiding missed lines, so the two metrics must be read together. Branch and eval coverage lean on newer `Coverage` features, so they carry Ruby-version floors.
- **Filters and groups.** Four default filters run on `start`: `root_filter` (drops files outside `SimpleCov.root`, so you don't get coverage for every gem), `bundler_filter` (`/vendor/bundle/`), `hidden_filter` (dotfile paths — which also silently hides legitimate `.scripts/`-style dirs), and `test_frameworks` (`test/`, `spec/`, `features/`, `autotest/`). Filters and groups share one matcher grammar: a String is a path-substring match, plus Regexp, block, and Array forms.
- **Merging.** Each run writes `coverage/.resultset.json`. SimpleCov merges resultsets so a report reflects the whole suite, keyed by a command name (`RSpec`, `Cucumber`, etc.) it guesses from the runner. `SimpleCov.collate` combines resultsets from separate processes — the supported path for parallel CI shards.
- **Formatting.** Formatters are pluggable (`SimpleCov::Formatter::MultiFormatter` chains several). Third-party formatters — LCOV, Cobertura, Console — are community gems.

## Production Notes

- **Load order is everything.** If coverage numbers look impossibly low or a whole directory reads as 0%, the cause is almost always that `SimpleCov.start` ran after the code was required. For a Rails server under system/Selenium tests, you must require SimpleCov inside the server process (top of `bin/rails`, guarded by `RAILS_ENV == 'test'`), not just the test process.
- **The 0.18 merge change.** Historically SimpleCov merged old resultsets by timeout, so stale results from a previous run could silently inflate today's number. Version 0.18 (2020) removed that cross-run auto-merge and pushed parallel workflows toward explicit `collate`. This broke many CI setups that had relied on the implicit behavior, and remains the biggest "why did my coverage drop after upgrading" story. `merge_timeout` still governs how long a resultset stays eligible within the merge window.
- **Parallel / sharded CI.** Have each shard write its resultset (often with `formatter false` to skip per-job HTML), collect the artifacts, then run one `SimpleCov.collate` pass to produce the final report and threshold check. Running per-shard thresholds instead produces false failures because no single shard sees the whole codebase.
- **`minimum_coverage` and exit codes.** SimpleCov sets a non-zero process exit status when a threshold is missed, which is what fails CI. Because it works via `at_exit`, another `at_exit` or an explicit `exit!` in your suite can swallow it — verify the build actually fails when coverage drops.
- **eval-generated noise.** Rails `delegate` and other `module_eval`-based macros make `Coverage` attribute synthetic methods/branches to the macro's line, showing as missed. Newer SimpleCov can strip these (`ignore_methods`/`ignore_branches :eval_generated`) using Prism to walk the source, but on older Rubies without Prism it's a silent no-op.
- **Templates aren't covered.** ERB/Haml/Slim are not tracked by default; only eval coverage on ERB (with `ERB#filename=` set) gets partial visibility. Don't expect view coverage.
- **Config API in flux.** The long-stable verbs are `add_filter`, `add_group`, `track_files`, `minimum_coverage`. A redesigned grammar (`skip`, `group`, `cover`, `coverage(:line) { ... }`, `merge_subprocesses`) is landing on `main` with deprecation warnings mapping the old names; pin your gem version and read the CHANGELOG before upgrading a shared config.

## When to Use / When Not

**Use when:**
- Any Ruby test suite where you want a coverage number and a browsable report with near-zero setup.
- Rails apps — the bundled `rails` profile and Cucumber/RSpec merging are the intended happy path.
- CI gating on a minimum threshold, per-file or per-group.

**Avoid / look elsewhere when:**
- You need view-template (ERB/Haml/Slim) coverage — largely out of scope.
- You want coverage of a non-Ruby or polyglot codebase — SimpleCov only understands what Ruby's `Coverage` sees.
- You're on a Ruby old enough to lack the `Coverage` features you need (branch/eval coverage have version floors).

## Alternatives

- deininger/undercover — reports coverage only for lines changed in a diff, using SimpleCov's resultset; complements rather than replaces it.
- danmayer/coverband — production/runtime coverage (what code real traffic exercises), a different question than test coverage.
- fortissimo1997/simplecov-lcov — a formatter, not a replacement; use when your CI (Coveralls, Codecov) wants LCOV output from SimpleCov.
- codecov / coveralls hosted services — consume SimpleCov's output for PR-level reporting and trend graphs; they sit above SimpleCov, not beside it.
- Ruby stdlib `Coverage` directly — use when you want raw counts with zero dependencies and will build your own reporting.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2010 | Initial release, built on Ruby 1.9's new `Coverage` stdlib[^1]. |
| 0.9.x | 2014 | Widely-adopted 0.9 series; long-lived config API (`add_filter`/`add_group`). |
| 0.18.0 | 2020-02 | Branch/method coverage, `collate`; removed cross-run auto-merge (major migration break)[^2]. |
| 0.21.0 | 2021-01 | Bugfixes and formatter refinements over the 0.18 line. |
| 0.22.0 | 2023 | HTML and JSON formatters vendored in; continued 0.2x maintenance line. |

## References

[^1]: Ruby standard library `Coverage` module — the engine SimpleCov wraps. https://docs.ruby-lang.org/en/master/Coverage.html
[^2]: SimpleCov CHANGELOG. https://github.com/simplecov-ruby/simplecov/blob/main/CHANGELOG.md

## Tags

ruby, code-coverage, testing, test-coverage, ci, rails, rspec, minitest, developer-tools, coverage-report
