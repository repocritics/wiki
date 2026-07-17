# muellan/clipp

> Header-only command-line argument parser for C++11 built around an operator-overloading DSL that generates both the parser and its documentation from a single declaration.

[GitHub repo](https://github.com/muellan/clipp) ·
[License: MIT](https://github.com/muellan/clipp/blob/master/LICENSE)

## Overview

clipp is a single-header C++11 library for parsing command-line arguments, first published by André Müller in 2017[^1]. Its distinguishing idea is that the command-line interface is written as a C++ expression: parameters (`option`, `required`, `value`, `command`, ...) are combined with overloaded operators — `,` for compatible groups, `|` for alternatives, `&` for sequences, `%` for documentation strings — into a parse tree that is both the grammar and the source for generated usage lines, man pages, and detailed docs. One declaration drives parsing and documentation, so the two cannot drift apart.

The DSL is the library's defining tension. It expresses genuinely complex interfaces — nested alternatives, decision trees, repeatable groups, joinable flags, greedy values — far more compactly than the callback-registration style of most parsers. In exchange it leans heavily on operator overloading and expression templates, which means malformed interfaces produce long template error messages, operator precedence (`,` vs `&` vs `|`) has semantics that are easy to get subtly wrong, and the ordering rules around "blocking" (positional) parameters take real study to predict. clipp is a tool that rewards reading the manual in full before writing anything nontrivial.

As of 2026 the repository has roughly 1,300 stars and 150 forks, but the last commit landed 2024-05-30 and there are 56 open issues[^2]. It is best treated as feature-complete-and-dormant rather than actively developed: stable and widely used, but do not expect bug fixes or responses upstream.

## Getting Started

There is no build step or package to install — copy the single `clipp.h` into your include path (also available via vcpkg and Conan as `clipp`, and packaged in some distro repos).

```cpp
#include <iostream>
#include "clipp.h"
using namespace clipp;

int main(int argc, char* argv[]) {
    bool recursive = false, utf16 = false;
    std::string infile, fmt = "csv";

    auto cli = (
        value("input file", infile),
        option("-r", "--recursive").set(recursive).doc("convert recursively"),
        option("-o") & value("output format", fmt),
        option("-utf16").set(utf16).doc("use UTF-16 encoding")
    );

    if (!parse(argc, argv, cli))               // excludes argv[0]
        std::cout << make_man_page(cli, argv[0]);
}
```

The same `cli` object is passed to `parse()`, `usage_lines()`, `documentation()`, and `make_man_page()`. Targets bound with `.set()` / `.call()` must be a fundamental type, something convertible from `const char*`, or a callable taking zero or one such argument.

## Architecture / How It Works

Everything is assembled at compile time. Factory functions (`option`, `value`, `command`, `required`, `repeatable`, `joinable`, `with_prefix`, ...) return `parameter` and `group` objects; the overloaded operators nest them into a tree. A `parameter` carries its flag strings or a match filter (`match::nonempty`, `word`, `number`, `integer`, or a custom predicate) plus flags for required / repeatable / blocking and any bound action.

Parsing is a single left-to-right walk of `argv` against this tree, driven by a small state machine. The key concept is **blocking** (positional) parameters: a blocking parameter is a stop point — until it matches, nothing after it in its group may match; once it matches, everything before it in that group becomes unreachable. Non-blocking parameters (ordinary flags) may match in any order. Alternatives (`|`) select one branch; sequences (`&` / `in_sequence`) force order within an otherwise order-free surrounding group. This is what lets clipp model git-style subcommand trees, but it is also the source of most "why didn't my argument match" confusion.

The result object from `parse()` aggregates the walk: `any_error()`, `unmapped_args_count()`, `any_conflict()`, `any_blocked()`, `any_bad_repeat()`, plus a `missing()` list and a per-argument mapping. Per-parameter event handlers (`if_missing`, `if_repeated`, `if_blocked`, `if_conflicted`) fire during the walk and can take the argument index. Documentation generation is a separate traversal of the same tree, formatted through a `doc_formatting` struct (column positions, line widths).

There are no dependencies and no runtime allocation model beyond standard containers; the whole library is one header of several thousand lines, all in `namespace clipp`.

## Production Notes

**The project is dormant.** Last commit 2024-05-30, 56 open issues, and the README still carries Travis CI / AppVeyor badges that point at now-defunct services[^2]. Known parser edge cases (particularly around joinable flags, greedy `!value`, and repeatable nested groups) sit unfixed in the issue tracker. Adopt it for what it is today; budget for maintaining your own fork if you hit a bug.

**Compile-time cost.** clipp is template- and operator-heavy, and `clipp.h` is large. Including it in many translation units, and building large interfaces, measurably increases compile times. The common mitigation is to confine CLI definition to a single `.cpp` that returns a plain settings struct, so the header is parsed once rather than per-TU.

**Error messages.** Because the interface is an expression template, a mistake in the DSL (wrong operator, a missing target overload, an ill-formed group) surfaces as a deep template diagnostic pointing inside `clipp.h`, not at your line. Build interfaces incrementally.

**Precedence and ordering footguns.** `,`, `&`, and `|` have C++'s native operator precedence, which does not always match the grouping you intend — parenthesize deliberately. The blocking/positional semantics mean the *order* in which you list parameters changes what parses; rearranging a group is not a no-op. Greedy values (`!value(...)`) override normal matching priority and should be used sparingly.

**No shell completion, no environment-variable binding, no config-file layer.** clipp parses `argv` and generates docs; anything beyond that (bash/zsh completion, `--` passthrough conventions, subcommand help routing) is left to the caller.

## When to Use / When Not

**Use when:**
- You want the interface, the parser, and the generated man page to come from one declaration.
- Your CLI has genuinely complex structure — nested subcommands, alternatives, decision trees, joinable flags — that a flat flag registry expresses poorly.
- You want a drop-in single header with no dependency or build integration, on a C++11 baseline.

**Avoid when:**
- You need an actively maintained library with responsive upstream (clipp is dormant).
- Your team is uncomfortable debugging expression-template error messages, or you want a gentle learning curve.
- You need shell completion, env-var/config binding, or a large modern-C++ (C++17/20) feature set out of the box.
- Compile time is a hard constraint and the CLI is large.

## Alternatives

- CLIUtils/CLI11 — actively maintained header-only parser with readable runtime error messages and subcommand support; use instead when upstream health and diagnostics matter more than DSL expressiveness.
- p-ranav/argparse — C++17, models Python's argparse API; use when you want a familiar, less template-heavy style.
- jarro2783/cxxopts — lightweight header-only option parser; use for simple flag/value CLIs without subcommand trees.
- docopt/docopt.cpp — define the interface by writing the help text; use when the usage string *is* your preferred source of truth.
- Taywee/args — single-header, argparse-inspired; use as a middle ground between cxxopts's simplicity and clipp's expressiveness.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Initial public release | 2017-11-08 | Single-header C++11 DSL, doc generation[^1]. |
| v1.2.x release line | 2018–2019 | Tagged bug-fix releases (exact dates vary by tag). |
| Last commit | 2024-05-30 | Repository dormant since; 56 open issues[^2]. |

## References

[^1]: muellan/clipp README and repository — André Müller, initial commit 2017-11-08. https://github.com/muellan/clipp
[^2]: GitHub API metadata for muellan/clipp (stars ~1,322, forks ~154, 56 open issues, last push 2024-05-30, MIT, C++), retrieved 2026-07. https://github.com/muellan/clipp

## Tags

cpp, cpp11, command-line, argument-parser, cli, header-only, single-header, dsl, man-page, dormant
