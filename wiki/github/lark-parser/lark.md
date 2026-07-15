# lark-parser/lark

> A pure-Python parsing toolkit that generates parse trees automatically, letting you pick between an Earley parser (any context-free grammar, ambiguity-tolerant) and LALR(1) (fast, restrictive) from the same grammar file.

[GitHub repo](https://github.com/lark-parser/lark) ·
[Documentation](https://lark-parser.readthedocs.io/) ·
[License: MIT](https://github.com/lark-parser/lark/blob/master/LICENSE)

## Overview

Lark is a parsing library for Python written by Erez Shinan, first published in 2017[^1]. You give it an EBNF-style grammar and an input string; it returns an annotated parse tree without any tree-construction code on your side. The defining design choice is that one grammar dialect drives two very different parsing engines: an **Earley** parser that handles every context-free grammar (including ambiguous ones) via a Shared Packed Parse Forest, and an **LALR(1)** parser that is fast and memory-light but rejects grammars with conflicts. You switch between them with a single `parser=` argument.

That duality is also the central tension. Earley is the default because it "just works" on almost any grammar, but it is O(n³) in the worst case and memory-heavy on large inputs. LALR is the parser you actually want in production, but moving a grammar from Earley to LALR often surfaces shift/reduce and reduce/reduce conflicts that force you to restructure the grammar or hand-tune terminal priorities. Much of the real-world skill in using Lark is knowing which engine a given grammar belongs to.

Lark is pure Python with no required dependencies, which makes it trivial to vendor and lets it run on any interpreter (CPython, PyPy). It is widely embedded: Poetry's core, Vyper, Hypothesis's grammar tooling, and many DSLs and config-file parsers build on it[^2]. As of 2026 it is one of the most-used parsing libraries in the Python ecosystem, actively maintained on `master` with releases continuing into the 1.x line.

## Getting Started

```bash
pip install lark --upgrade
```

Note the package name is `lark` (renamed from `lark-parser` at v1.0); the import has always been `import lark`.

```python
from lark import Lark

grammar = r"""
    start: WORD "," WORD "!"
    %import common.WORD    // standard-library terminal
    %ignore " "            // skip spaces
"""

parser = Lark(grammar)                      # Earley by default
tree = parser.parse("Hello, World!")
print(tree)   # Tree('start', [Token('WORD','Hello'), Token('WORD','World')])
```

Processing a tree with a `Transformer` (bottom-up rewrite):

```python
from lark import Lark, Transformer

json_parser = Lark(r"""
    value: dict | list | ESCAPED_STRING | SIGNED_NUMBER | "true" | "false" | "null"
    list : "[" [value ("," value)*] "]"
    dict : "{" [pair ("," pair)*] "}"
    pair : ESCAPED_STRING ":" value
    %import common (ESCAPED_STRING, SIGNED_NUMBER, WS)
    %ignore WS
""", start="value", parser="lalr")           # LALR: fast, table-based

class T(Transformer):
    def list(self, items): return list(items)
    def SIGNED_NUMBER(self, tok): return float(tok)

print(T().transform(json_parser.parse('[1, 2, 3]')))   # [1.0, 2.0, 3.0]
```

## Architecture / How It Works

A `Lark` object is built in two phases. At construction time Lark loads the grammar, resolves `%import`s, expands EBNF operators (`*`, `+`, `?`, `[]`), and — for LALR — runs grammar analysis to build parse tables. That table construction is the expensive part; the `cache=True` option memoizes it to disk. At `parse()` time the lexer tokenizes and the chosen engine builds the tree.

**Parsers.** Three algorithms ship, but two matter:
- **Earley** (default) — handles all CFGs, supports rule/terminal priorities, and represents ambiguity as an SPPF that is collapsed to a single tree (`ambiguity='resolve'`) or exposed as `_ambig` forest nodes (`ambiguity='explicit'`).
- **LALR(1)** — a classic table-driven parser. No ambiguity, no conflicts allowed. It supports a `transformer=` passed directly to `Lark(...)` so the tree is transformed during parsing and never fully materialized — the memory-efficient path.
- **CYK** — present but rarely used; treat it as legacy.

**Lexers** are a separate axis from the parser. LALR uses a **contextual** lexer by default (it consults the parser state to decide which terminals are valid, which resolves many keyword-vs-identifier collisions). Earley defaults to a **dynamic** lexer that can try multiple tokenizations. A plain `basic` lexer is also available. Terminal matching is priority-then-length; string terminals outrank regex terminals of equal priority. Mis-tokenization from terminal collisions is the most common early confusion.

**Trees.** Output is `Tree(data, children)` and `Token` leaves. You post-process with `Transformer` (bottom-up, can replace subtrees), `Visitor` (bottom-up, side effects), or `Interpreter` (top-down, you control recursion). `v_args` controls how children are passed to callbacks. `propagate_positions=True` attaches line/column metadata.

**Standalone generation.** `python -m lark.tools.standalone grammar.lark > parser.py` emits a single-file LALR parser with no Lark runtime dependency — useful for shipping a parser without a `pip` dependency. The standalone tool is licensed MPL-2.0, distinct from the library's MIT[^3]. Lark can also import Nearley.js grammars.

## Production Notes

**Pick LALR for anything hot.** Earley is a great prototyping and "unknown grammar" tool, but for a fixed grammar on non-trivial inputs it is the wrong default in production — slower and far more memory-hungry. The practical workflow is: develop with Earley, then port to LALR and pay down the conflicts. Grammars with genuine ambiguity (many DSLs, most natural-language work) cannot make that move and stay on Earley.

**Conflict messages are terse.** LALR reduce/reduce and shift/reduce errors point at grammar symbols, not your intent. Fixes are usually terminal `%priority` adjustments, rule restructuring, or leaning on the contextual lexer. Budget time for this the first time you convert a real grammar.

**Construction cost.** Building the `Lark` object (table generation) can dominate for large LALR grammars. Use `cache=True` (or a cache filename) so repeated process starts reuse tables, or ship a standalone parser to skip analysis entirely.

**`maybe_placeholders` default flipped at 1.0.** Optional `[...]` items now yield `None` placeholders in the tree by default, so child positions stay stable. Grammars written against 0.x tree shapes can break silently on upgrade — set `maybe_placeholders=False` to restore old behavior, or update your transformers.

**Package rename.** `lark-parser` on PyPI is the frozen pre-1.0 name; new installs use `lark`. Having both installed in one environment causes import shadowing — pin one.

**Error handling / recovery.** Lark raises `UnexpectedToken` / `UnexpectedCharacters` with a helpful context window and `expected` set. There is no automatic error-recovery mode; for editor-style tolerant parsing you use the **interactive parser** (`parse_interactive`) or an `on_error` callback with LALR to feed synthetic tokens. Incremental/streaming reparse is not built in.

**Unicode & regex.** Terminals are Python regexes. The standard `re` module is used unless you install the optional `regex` package, which enables advanced Unicode property classes in terminals.

## When to Use / When Not

**Use when:**
- You have a grammar and want a parse tree without writing tree-building code.
- You need to parse a real DSL, config format, or query language and want to prototype fast (Earley) then harden (LALR) without rewriting the grammar.
- You need ambiguity handling or "parse any CFG" guarantees that PEG/combinator libraries cannot give.
- You want zero dependencies and the option to emit a standalone single-file parser.

**Avoid when:**
- You need parsers in multiple target languages from one grammar — use ANTLR.
- Your input is trivial (a few tokens, a simple regex or split would do) — a full grammar is overkill.
- You need maximum raw throughput on huge inputs in a compiled language — a hand-written or generated C/Rust parser will beat pure-Python Lark.
- You want built-in error recovery for an IDE without assembling it yourself.

## Alternatives

- pyparsing/pyparsing — PEG-style combinators defined inline in Python; use it when you prefer no separate grammar file and don't need ambiguity or full CFG coverage.
- antlr/antlr4 — industrial LL(*) generator with codegen for many languages; use it when you need the same grammar to produce Java/Go/JS/Python parsers or want mature IDE tooling.
- dabeaz/sly — a modern lex/yacc for Python (successor to PLY); use it when you want the traditional token-rules + LALR model with explicit control.
- dabeaz/ply — the classic yacc/lex-in-Python; use it for legacy compatibility or teaching the yacc workflow.
- erikrose/parsimonious — small, fast PEG parser with an EBNF-ish grammar; use it when a deterministic PEG is enough and you want minimal surface area.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-02 | First public release as `lark-parser`[^1]. |
| 0.8.0 | 2020-01 | Major grammar/lexer refinements during the 0.x line. |
| 0.12.0 | 2021-08 | Last significant 0.x release before the 1.0 cutover. |
| 1.0.0 | 2021-11-15 | Package renamed to `lark`, Python 2 dropped, `maybe_placeholders` default changed[^4]. |
| 1.1.0 | 2022-01-31 | Continued 1.x feature and typing work. |
| 1.2.0 | 2024-08 | 1.2 line. |
| 1.3.0 | 2025-09-22 | Latest minor line[^5]. |

## References

[^1]: Lark repository, created 2017-02-04. https://github.com/lark-parser/lark
[^2]: "Projects using Lark", README dependents list. https://github.com/lark-parser/lark#projects-using-lark
[^3]: Lark README — "The standalone tool is under MPL2." https://github.com/lark-parser/lark#license
[^4]: Lark 1.0.0 release, 2021-11-15. https://github.com/lark-parser/lark/releases/tag/1.0.0
[^5]: Lark 1.3.0 release, 2025-09-22. https://github.com/lark-parser/lark/releases/tag/1.3.0

## Tags

python, parser, parsing, earley, lalr, grammar, ebnf, parser-generator, dsl, ast, compiler-tools, pure-python
