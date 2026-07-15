# tobymao/sqlglot

> A dependency-free Python library that parses SQL into an editable AST and transpiles it between ~31 dialects.

[GitHub repo](https://github.com/tobymao/sqlglot) ·
[Official website](https://sqlglot.com/) ·
[License: MIT](https://github.com/tobymao/sqlglot/blob/main/LICENSE)

## Overview

SQLGlot is a SQL parser, transpiler, optimizer, and toy execution engine written in
pure Python with no runtime dependencies. Its core value proposition is round-tripping:
it reads SQL from one dialect (Spark, BigQuery, Snowflake, DuckDB, Presto/Trino, T-SQL,
and roughly two dozen others) into a normalized expression tree, then regenerates
syntactically and semantically adjusted SQL for a target dialect. It also formats,
detects syntax errors, and lets you build and mutate queries programmatically instead of
doing string surgery.

The project is maintained primarily by Toby Mao and is closely tied to Tobiko Data, where
SQLGlot is the parsing foundation under SQLMesh[^2]. That lineage matters: SQLGlot is
built to be a library other data tools embed, not an end-user CLI. It is depended on by
Apache Superset, Ibis, Dagster, dlt, Fugue, and Splink among others, which is why its
correctness bar is unusually high for a solo-led project.

The defining tension is scope versus stability. SQLGlot aims to be a *superset* of every
dialect it supports, and transpilation between arbitrary dialect pairs is an open-ended,
"incremental" problem — some input/output combinations simply are not covered yet, and
the maintainer is explicit that this improves over time rather than being complete[^1].
Paired with an aggressive versioning policy (below), this makes SQLGlot excellent at what
it covers and demanding to pin in production.

## Getting Started

```bash
pip install sqlglot          # pure Python
pip install "sqlglot[c]"     # mypyc-compiled C extensions, ~3-4x faster parsing
```

```python
import sqlglot

# Transpile: read one dialect, write another. Dialects are mandatory for correctness.
print(sqlglot.transpile("SELECT EPOCH_MS(1618088028295)", read="duckdb", write="hive")[0])
# 'SELECT FROM_UNIXTIME(1618088028295 / POW(10, 3))'

# Parse into an AST and introspect
from sqlglot import parse_one, exp
tree = parse_one("SELECT a, b + 1 AS c FROM d", dialect="snowflake")
for col in tree.find_all(exp.Column):
    print(col.alias_or_name)          # a, b
```

## Architecture / How It Works

The pipeline is Tokenizer → Parser → Expression tree → Generator. The tree is a hierarchy
of `exp.Expression` nodes (`Select`, `Column`, `Add`, `Literal`, `Identifier`, …); every
transformation in SQLGlot is a tree rewrite, and `.sql(dialect=...)` walks the tree to
emit text. There is no separate IR — the AST *is* the intermediate representation, which
is why introspection (`find_all`, `.transform`, `diff`) and generation share one model.

Dialects are subclasses of `Dialect` that override `Tokenizer` (keyword and quote-char
maps), `Parser`, and `Generator` (`TRANSFORMS`, `TYPE_MAPPING`). When you omit a dialect,
SQLGlot uses its own permissive "SQLGlot dialect" designed to be a superset of all
supported grammars — which is why a query that parses without `read=` may still fail to
round-trip to a real target. Custom dialects are a first-class extension point: subclass,
override the maps, and the new dialect registers itself.

The **optimizer** (`sqlglot.optimizer`) is a separate, optional layer of rule passes:
`qualify` (resolve column/table references against a schema), `annotate_types` (type
inference), predicate pushdown, constant folding, and canonicalization. These rules need a
`schema=` argument to do type-sensitive work and are *not* run during ordinary
transpilation because they add significant overhead — a deliberate split between "translate
the text" and "understand the semantics." On top of the optimizer sits `sqlglot.executor`,
a pure-Python engine that runs queries over Python dicts; it is explicitly a correctness/
teaching tool, not a fast engine, though it is designed to be swappable with Arrow/Pandas
kernels.

Performance-critical paths are compiled with mypyc into C extensions, shipped as the
`sqlglot[c]` extra. Benchmarks in-repo put the compiled build at roughly 3-4x the pure-
Python parse speed, landing it in the same order of magnitude as the Rust-binding parsers
(sqloxide, polyglot-sql) despite being Python[^3].

## Production Notes

- **Minor versions break APIs by design.** SQLGlot's stated policy increments the MINOR
  version for backwards-*incompatible* fixes and features, and MAJOR for significant
  ones[^1]. Combined with a fast release cadence (the project has moved through 28+ major
  lines since 2021), this means `sqlglot>=X` in a requirements file is a real hazard. Pin
  exact versions and test transpilation output on upgrade — generated SQL can change subtly
  between releases as dialect coverage improves.
- **Always pass the source dialect.** The single most common bug report is "valid SQL fails
  to parse," and the fix is almost always supplying `read=`/`dialect=`. The default SQLGlot
  dialect is a permissive superset and will mis-handle dialect-specific syntax silently.
- **Unsupported translations degrade quietly.** By default an untranslatable construct emits
  a warning and does a best-effort conversion (e.g. dropping `APPROX_DISTINCT` accuracy). Set
  `unsupported_level=ErrorLevel.RAISE` in pipelines where a lossy translation must fail loudly.
- **Type-sensitive transforms require a schema.** Correct transpilation of some constructs
  depends on knowing column types; without `qualify`/`annotate_types` and a `schema=`, those
  cases fall back to best-effort. Feeding schemas is the difference between "mostly right" and
  "correct."
- **`sqlglot.dataframe` was removed in v24**, extracted into the standalone SQLFrame project.
  Code importing the old PySpark-style DataFrame API will not run on modern versions[^4].
- **Pure Python has a floor.** For very large `IN` lists, `VALUES` blocks, or bulk parsing,
  even the compiled build is meaningfully slower than Rust parsers; if you parse millions of
  statements, budget CPU accordingly or use the `[c]` extra.

## When to Use / When Not

**Use when:**
- You need to translate SQL between dialects, or normalize/format queries programmatically.
- You want to analyze, lint, or rewrite SQL by manipulating an AST rather than regexes.
- You are building data tooling (lineage, query rewriting, a dialect-agnostic layer) and want
  an embeddable, dependency-free parser.

**Avoid when:**
- You only need lightweight formatting or statement splitting — a smaller tokenizer is enough.
- You need a validating parser bound to exactly one engine's grammar with guaranteed fidelity;
  that engine's own parser will always be more precise.
- You require a stable, rarely-changing API surface — SQLGlot's minor-breaks policy fights that.
- You want a production query engine — the built-in executor is for testing, not workloads.

## Alternatives

- andialbrecht/sqlparse — use when you only need non-validating tokenizing, splitting, or
  reformatting in Python and don't need dialect translation or a real AST.
- sqlfluff/sqlfluff — use when the goal is linting and auto-fixing SQL style in CI, not
  programmatic transpilation (it actually uses a dialect model of its own).
- apache/calcite — use on the JVM when you need a mature, cost-based query optimizer and
  planner rather than a text-to-text transpiler.
- JSQLParser/JSqlParser — use on the JVM for straightforward parse/build of SQL without
  cross-dialect generation.
- sqloxide (Rust `sqlparser-rs` bindings) — use when raw parse throughput dominates and you
  don't need SQLGlot's transpilation, optimizer, or Python-native tree editing.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2021-03 | Initial public release by Toby Mao; pure-Python parser + transpiler[^1]. |
| v24 | 2024 | PySpark-style `sqlglot.dataframe` API extracted to the standalone SQLFrame project[^4]. |
| v28.x | 2026 | Current line; mypyc-compiled `sqlglot[c]` wheels, ~31 supported dialects[^3]. |

## References

[^1]: SQLGlot README — overview, versioning policy, and "incremental" transpilation stance. https://github.com/tobymao/sqlglot/blob/main/README.md
[^2]: SQLMesh (Tobiko Data), which embeds SQLGlot as its SQL parsing/transpilation layer. https://github.com/TobikoData/sqlmesh
[^3]: SQLGlot benchmarks (`benchmarks/parse.py`) comparing pure-Python, `sqlglot[c]`, and Rust-binding parsers. https://github.com/tobymao/sqlglot/blob/main/benchmarks/parse.py
[^4]: SQLFrame — the DataFrame API split out of SQLGlot in v24. https://github.com/eakmanrq/sqlframe

## Tags

python, sql, parser, transpiler, sql-optimizer, ast, dialect-translation, data-engineering, query-analysis, mypyc, no-dependencies
