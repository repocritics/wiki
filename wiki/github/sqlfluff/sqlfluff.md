# sqlfluff/sqlfluff

> A dialect-aware SQL linter and auto-formatter that understands Jinja and dbt templating.

[GitHub repo](https://github.com/sqlfluff/sqlfluff) ·
[Official website](https://www.sqlfluff.com) ·
[License: MIT](https://github.com/sqlfluff/sqlfluff/blob/main/LICENSE.md)

## Overview

SQLFluff is a configurable SQL linter and auto-formatter written in Python, first
released in 2019 by Alan Cruickshank[^1]. Its distinguishing feature is that it lints
*templated* SQL, not rendered SQL. It compiles Jinja, dbt, or placeholder templates to
raw SQL, checks the compiled tree, then maps every proposed fix back to the template
source so the edit lands in your `.sql` model, not the rendered output. That
template-position mapping is the hard problem SQLFluff exists to solve, and the reason
it became the de facto linter for the dbt/ELT ecosystem rather than one of the many
general SQL formatters.

SQLFluff is built for analytics engineers and data teams enforcing a house SQL style
across a warehouse codebase (BigQuery, Snowflake, Redshift, Postgres, and ~25 other
dialects). It ships two commands: `lint` (report violations) and `fix` (auto-apply
the subset of rules that can safely rewrite), with many rules — layout, whitespace,
capitalisation, aliasing — auto-fixable.

The central tension is coverage versus correctness. SQL dialects are large and
SQLFluff's grammars are hand-written, so any dialect is "supported but not complete."
When the parser meets syntax it does not know, it emits *unparsable* segments and
silently skips rule enforcement in that region — a green lint on code the linter did
not understand. Combined with the version coupling of the dbt templater, this makes
SQLFluff capable but operationally fiddly.

## Getting Started

```shell
pip install sqlfluff
echo "SELECT a  +  b from tbl" > test.sql
sqlfluff lint test.sql --dialect ansi
sqlfluff fix  test.sql --dialect ansi     # rewrites the file in place
```

Configuration lives in a `.sqlfluff` file (INI-style) or under
`[tool.sqlfluff]` in `pyproject.toml`. A `dialect` is mandatory:

```ini
# .sqlfluff
[sqlfluff]
dialect = snowflake
templater = jinja
exclude_rules = LT05, RF01
```

For dbt projects, install the separate templater plugin and point the config at it:

```shell
pip install sqlfluff sqlfluff-templater-dbt   # version must track your dbt-core
```

## Architecture / How It Works

The pipeline is five stages, run per file:

1. **Templater** — renders Jinja / dbt / placeholder / Python-format templates into raw
   SQL, producing a `TemplatedFile` that records a slice-by-slice map between templated
   and source positions. This map is what lets a fix computed on rendered SQL be written
   back to the template.
2. **Lexer** — splits the rendered string into raw tokens.
3. **Parser** — matches tokens against a *dialect grammar* (Python objects like
   `Sequence`, `OneOf`, `Bracketed`, `Delimited`) to build a tree of `segments`.
   Dialects inherit from `ansi` and override grammar, so Snowflake is "ANSI plus
   overrides," Databricks extends SparkSQL, and so on.
4. **Linter (rules)** — each rule is a class with a code that crawls the segment tree,
   emitting `LintResult`s and optional `LintFix` edits.
5. **Fixer** — applies non-conflicting fixes, re-parses, and iterates until stable.

Rules are grouped by a two-letter prefix: `LT` layout, `CP` capitalisation, `AL`
aliasing, `AM` ambiguous, `CV` convention, `RF` references, `ST` structure, `TQ`
T-SQL, `JJ` Jinja. The **2.0 release (2023) renamed the entire rule set** from opaque
codes (`L001`–`L066`) to these named references[^2] — a hard break for every CI config
and inline `noqa` comment that referenced the old numbers.

Since 4.0 (2026) an **optional Rust parser and lexer** is available via the `sqlfluff[rs]`
extra[^3]. It is opt-in beta, materially faster on large files and whole projects, and
slightly slower on small individual files due to marshalling overhead. The maintainers
have signalled Rust becoming the default from 5.0.

## Production Notes

**The dbt templater is the main operational cost.** It works by actually invoking dbt
to compile your models, so it needs the dbt project, a valid `profiles.yml`, and an
installed adapter — and `sqlfluff-templater-dbt` is version-coupled to `dbt-core`. A
warehouse upgrade or dbt bump can break linting until you re-pin. Compilation also
makes it the slowest templater by a wide margin; large projects lint in minutes, not
seconds.

**Silent gaps from unparsable SQL.** Because grammars are incomplete per dialect,
unknown syntax parses as unparsable and rules are skipped there — a passing lint is not
proof the linter understood the file. Run `sqlfluff parse` on suspicious files and watch
for `unparsable` sections. Raising the missing-grammar issue upstream is usually the fix.

**Templating false positives.** Complex Jinja (loops that build column lists, macros
that emit partial statements) can render to SQL whose structure the position map can't
cleanly attribute, producing spurious `TMP`/`PRS` errors or fixes that refuse to apply.
The `dbt` templater is more accurate than the generic `jinja` one for dbt code because
it uses dbt's own compilation.

**`fix` is a rewriter — keep it in version control.** Most fixes are layout-safe, but
run `fix` on a clean git tree and diff before committing; `--fixed-suffix` writes to a
new file if you want to inspect first.

**Config precedence.** `.sqlfluff` files are read hierarchically from the file upward,
so nested directories can tighten or relax rules; `exclude_rules`, `rules`, and per-rule
sections interact in ways worth testing with `--verbose`.

**Pin the version in CI.** Official pre-commit hooks and a GitHub Action are the common
integrations. Minor releases add and re-tune rules, so an unpinned upgrade can turn a
green pipeline red.

## When to Use / When Not

**Use when:**
- You maintain a dbt or Jinja-templated SQL codebase and want style enforced on the
  *source*, not the compiled output.
- You need per-dialect linting across BigQuery / Snowflake / Redshift / Postgres etc.
- You want auto-fix for layout, capitalisation, and aliasing in CI or pre-commit.

**Avoid when:**
- You want zero-config opinionated formatting and no rule tuning — an autoformatter
  like sqlfmt is faster to adopt.
- Your dialect's advanced syntax is only partially supported and you'd hit unparsable
  gaps (check `sqlfluff parse` on a sample first).
- You only need a SQL parser as a library, not a lint/fix CLI.

## Alternatives

- tconbeer/sqlfmt — opinionated, zero-config auto-formatter for dbt SQL ("Black for
  SQL"); use it when you want deterministic formatting and no lint rules to tune.
- andialbrecht/sqlparse — non-validating Python SQL parser with basic reindenting; use
  it as a library, not as a style linter.
- sbdchd/squawk — Postgres migration/DDL linter; use it for migration-safety checks
  (locks, unsafe column changes) rather than style enforcement.
- sql-formatter-org/sql-formatter — JavaScript multi-dialect SQL formatter; use it
  inside a Node toolchain where a Python dependency is unwanted.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2019 | First public release[^1]. |
| 1.0.0 | 2022-06-17 | First stable major; rule set and config surface stabilised. |
| 2.0.0 | 2023-03-14 | Rules renamed `L0xx` → named references (`LT01` etc.); dbt templater config overhaul[^2]. |
| 3.0.0 | 2024-03-12 | Dropped older Python/dbt versions; internals cleanup. |
| 4.0.0 | 2026-01-15 | Opt-in Rust parser/lexer via `sqlfluff[rs]`; dropped dbt ≤1.4[^3]. |

## References

[^1]: SQLFluff documentation and project history. https://docs.sqlfluff.com/en/stable/
[^2]: SQLFluff 2.0.0 release — rule reference rename and templater changes, 2023-03-14. https://github.com/sqlfluff/sqlfluff/releases/tag/2.0.0
[^3]: SQLFluff 4.0.0 release — optional Rust parser/lexer (`sqlfluff[rs]`), 2026-01-15. https://github.com/sqlfluff/sqlfluff/releases/tag/4.0.0

## Tags

python, sql, linter, formatter, dbt, jinja, static-analysis, data-engineering, cli, elt, code-quality
