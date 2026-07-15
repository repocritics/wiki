# presidentbeef/brakeman

> Static taint analysis for Ruby on Rails — finds injection and mass-assignment bugs from source, without running the app.

[GitHub repo](https://github.com/presidentbeef/brakeman) ·
[Official website](https://brakemanscanner.org/) ·
[License: Brakeman Public Use License](https://github.com/presidentbeef/brakeman/blob/main/LICENSE.md) (non-commercial; MIT for pre-2018 code)

## Overview

Brakeman is a static analysis security scanner specialized for Ruby on Rails
applications, created by Justin Collins in 2010[^1]. Unlike generic linters or
runtime scanners, it parses a Rails app's source into ASTs and traces how user
input (`params`, `cookies`, request data) flows into dangerous sinks — SQL
strings, `eval`, `render`, redirects, `system` calls, unsafe HTML. It ships
knowledge of Rails conventions (routes, controllers, models, ERB/Haml/Slim
views), so it can reason about a `link_to` or a `where(...)` the way a Rails
developer does. It runs against source only: no server, no database, no test
suite, and no code execution.

For most of its life Brakeman has been the default SAST tool in the Rails
ecosystem — used in CI at GitHub, Twitter, Groupon, and many others[^2]. Its
defining tension is licensing, not engineering. Code committed before 2018-06-15
is MIT; everything since is under the **Brakeman Public Use License**, a
non-commercial license, with copyright assigned to Synopsys, Inc[^3]. The gem
you `gem install` today is therefore not OSI-open-source, which surprises teams
who treat it as MIT and which is why GitHub's license detector reports
`NOASSERTION`.

## Getting Started

```bash
gem install brakeman
```

```ruby
# Gemfile — keep it out of the production bundle
group :development do
  gem 'brakeman', require: false
end
```

```bash
# From a Rails app root: scan and write a triage-friendly report
brakeman

# Machine-readable output for CI (format inferred from extension)
brakeman -o brakeman.json -o brakeman.sarif

# Fail CI only above a confidence level (3 = high confidence only)
brakeman -w3 -q
```

Brakeman runs a compatibility split worth knowing up front: it can analyze code
written in Ruby 2.0 syntax and Rails from 2.3.x through 8.x, but the scanner
itself requires **Ruby 3.2.0 or newer** to run[^4].

## Architecture / How It Works

Brakeman never executes the target app. Its pipeline is:

1. **Parse** — every `.rb`, `.erb`, `.haml`, etc. is turned into an AST via the
   `ruby_parser` gem, producing S-expressions Brakeman then post-processes into
   its own normalized form.
2. **Build a model of the app** — it walks `config/routes.rb`, controllers,
   models, and templates to understand which actions are reachable, what filters
   run, and how templates render. This Rails-awareness is what separates it from
   pattern-grep tools.
3. **Run checks** — each vulnerability class is a modular check (SQL injection,
   XSS, command injection, mass assignment, unsafe redirect, open `send`, weak
   crypto, and dozens more) that subclasses a common base and inspects the model.
4. **Track data flow** — checks follow assignments and method calls to decide
   whether attacker-controlled input actually reaches a sink, rather than just
   matching a dangerous method name.

Each finding carries a **confidence level** — High, Medium, or Weak — reflecting
how sure Brakeman is that input reaches the sink unsafely[^5]. This is the core
UX: it is a lead-generation tool for a human reviewer, not a pass/fail gate.
Output can be emitted as text, HTML (with source excerpts), JSON, SARIF, JUnit,
CodeClimate, CSV, markdown, or Sonar, so it slots into most CI and code-scanning
dashboards.

## Production Notes

**False positives are the workflow, not a bug.** Taint analysis over a dynamic
language is necessarily approximate. Custom sanitizers Brakeman doesn't
recognize produce false positives; metaprogramming and unusual DSLs it can't
resolve produce false *negatives*. The intended loop is to triage findings into
`config/brakeman.ignore` (managed interactively with `brakeman -I`) so that only
new warnings surface on subsequent runs. `--compare old_report.json` diffs two
scans into fixed/new lists, which is the standard CI pattern.

**It is unsound by design.** A clean Brakeman run is not proof of security. The
`--faster` flag (equivalent to `--skip-libs --no-branching`) trades coverage for
speed and will miss vulnerabilities; the docs say so explicitly. Even at full
strength, anything Brakeman cannot parse or model — heavy metaprogramming,
generated code, non-standard rendering — is simply invisible to it.

**Version drift against Rails.** Because checks encode knowledge of specific
Rails APIs and their safe/unsafe variants, new Rails releases periodically
introduce sinks or helpers Brakeman doesn't yet model. Keeping Brakeman current
matters as much as keeping Rails current; an old Brakeman on a new Rails app
quietly under-reports.

**Exit codes and noise.** By default Brakeman returns non-zero if any warning is
found *or* if it hits a parse/scan error — the latter can fail CI for reasons
unrelated to security. `--no-exit-on-warn` / `--no-exit-on-error` and
`--skip-files` are the escape hatches for unparseable files. All non-report
output goes to stderr, so `brakeman -o /dev/stdout` cleanly separates the report
from diagnostics.

**Licensing is an operational decision.** The current gem is non-commercial
under the Brakeman Public Use License with Synopsys copyright. Many companies use
it in commercial CI regardless; teams with strict license policies should read
`LICENSE.md` rather than assume MIT, since GitHub reports the license as
unrecognized.

## When to Use / When Not

**Use when:**
- You maintain a Rails app and want injection/XSS/mass-assignment findings
  without instrumenting or deploying anything.
- You want a fast source-only check in CI with a diffable, ignorable baseline.
- You need SARIF/JSON output to feed code-scanning dashboards.

**Avoid (or supplement) when:**
- Your app is not Rails — Brakeman's value is its Rails model; on Sinatra/plain
  Ruby it degrades to little.
- You need dependency-CVE coverage — that is `bundler-audit`'s job, not
  Brakeman's; the two are complementary.
- You need soundness/compliance guarantees — Brakeman finds bugs, it does not
  prove their absence.
- Non-commercial licensing is a blocker for your organization.

## Alternatives

- rubysec/bundler-audit — checks `Gemfile.lock` against known-vulnerable gem
  versions; complementary, use alongside Brakeman rather than instead of it.
- semgrep/semgrep — multi-language pattern SAST with custom rules; use when you
  need cross-language coverage or bespoke org-specific rules.
- github/codeql — semantic dataflow analysis including Ruby; use when you want
  deeper interprocedural analysis inside GitHub code scanning and accept heavier
  setup.
- thesp0nge/dawnscanner — older Ruby scanner also covering Sinatra/Padrino; use
  for non-Rails Ruby, noting slower maintenance.
- rubocop/rubocop — style/lint with a few security cops; use for general code
  quality, not as a security scanner substitute.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2010 | Initial release by Justin Collins; Rails-specific static scanner[^1]. |
| — | 2018-06-15 | License change: MIT → Brakeman Public Use License; copyright to Synopsys[^3]. |
| 5.x | ~2021 | SARIF output, broadened check set. |
| 6.x | ~2023 | Rails 7 coverage, dropped older Ruby runtimes. |
| 8.0.x | 2026 | Current line (v8.0.5); Rails 8 support, requires Ruby 3.2.0+[^4]. |

## References

[^1]: Brakeman project site and history. https://brakemanscanner.org/
[^2]: Brakeman README, "Who is Using Brakeman?" — GitHub, Twitter, Groupon, New Relic, Code Climate. https://github.com/presidentbeef/brakeman#who-is-using-brakeman
[^3]: Brakeman `COPYING.md` — MIT before 2018-06-15, Brakeman Public Use License after, Synopsys copyright. https://github.com/presidentbeef/brakeman/blob/main/COPYING.md
[^4]: Brakeman README, "Compatibility" — Rails 2.3.x–8.x, Ruby 2.0 syntax analyzable, Ruby 3.2.0+ required to run. https://github.com/presidentbeef/brakeman#compatibility
[^5]: Brakeman README, "Confidence levels" — High / Medium / Weak. https://github.com/presidentbeef/brakeman#confidence-levels

## Tags

ruby, ruby-on-rails, static-analysis, sast, security-scanner, taint-analysis, vulnerability-detection, appsec, cli, devsecops
