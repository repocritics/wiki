# jarro2783/cxxopts

> Header-only C++ command-line option parser that speaks GNU-style syntax and fits in a single include.

[GitHub repo](https://github.com/jarro2783/cxxopts) ·
[License: MIT](https://github.com/jarro2783/cxxopts/blob/master/LICENSE)

## Overview

cxxopts is a single-header C++ library for parsing command-line arguments in
the standard GNU getopt style (`--long`, `--long=value`, `-a`, bundled `-abc`).
It was started by Jarryd Beck in 2014[^1] and has become one of the default
answers to "I want argument parsing in C++ without pulling in Boost." The whole
library is one file, `cxxopts.hpp`, that you drop into your include path — no
build step, no link step, no dependency beyond a C++11 standard library.

The library's defining tradeoff is deliberate smallness. It covers the common
cases well — typed options via templates, positional arguments, vectors,
default and implicit values, grouped help output — and stops there. It has no
subcommand system (git-style `tool commit`), no config-file or environment
loading, and no error-code API: everything that goes wrong throws a C++
exception. For a program that wants a handful of flags parsed correctly with a
readable `--help`, that scope is a good fit. For a CLI with nested verbs and
shell-completion ambitions, it is a floor you will outgrow.

The second tension is that "header-only" is a distribution convenience, not a
free lunch. `cxxopts.hpp` pulls in `<regex>` and is parsed in full by every
translation unit that includes it, which shows up as compile-time and binary
cost in large projects (see Production Notes).

## Getting Started

Vendored (the common path): copy `include/cxxopts.hpp` into your tree. It is
also packaged by vcpkg, Conan, Homebrew, and most Linux distributions, and
ships a CMake `find_package(cxxopts)` target.

```cpp
#include <cxxopts.hpp>
#include <iostream>

int main(int argc, char** argv) {
  cxxopts::Options options("myprog", "One-line description");
  options.add_options()
    ("d,debug", "Enable debugging")                               // bool flag
    ("f,file", "Input file", cxxopts::value<std::string>())
    ("n,count", "Iterations", cxxopts::value<int>()->default_value("1"))
    ("h,help", "Print usage");

  auto result = options.parse(argc, argv);          // may throw on bad input

  if (result.count("help")) {
    std::cout << options.help() << std::endl;
    return 0;
  }
  bool debug = result["debug"].as<bool>();
  int  count = result["count"].as<int>();
  if (result.count("file"))
    std::cout << "file=" << result["file"].as<std::string>() << "\n";
  return 0;
}
```

Values are typed with `cxxopts::value<T>()`; any `T` with an `operator>>` can be
a target. Reading a missing or wrongly-typed option throws, so guard with
`result.count(...)` unless a `default_value` is set.

## Architecture / How It Works

Everything lives in `cxxopts.hpp` (a few thousand lines) inside the `cxxopts`
namespace. The design is three moving parts:

- **`Options`** — the schema builder. `add_options()` returns a helper whose
  `operator()` registers each option's short/long names, description, group, and
  a `Value` object describing how to parse and store it. `Value` is a templated
  wrapper (`values::standard_value<T>`) that captures `T` and does the actual
  string→value conversion.
- **The parser** — walks `argc`/`argv`, classifying each token as a long option,
  a short/bundled option, a `--` positional terminator, or a positional. String
  values are converted through templated `parse_value` helpers; integer and
  floating parsing use `std::regex` to validate format before conversion.
- **`ParseResult`** — a map from option name to the parsed value(s), plus the
  list of unmatched arguments. You query it with `count()` and `operator[]`,
  pulling typed values back out with `.as<T>()`.

Error handling is exception-only. Everything derives from
`cxxopts::exceptions::exception`; schema mistakes derive from `...::specification`
and bad user input derives from `...::parsing`, each with a `what()` string. There
is no `std::error_code`-style path — a program compiled with `-fno-exceptions`
cannot use the library.

Version 3 was a structural change to ownership. The parser no longer mutates
`argc`/`argv`, so you can pass `const` inputs, and `ParseResult` no longer holds
a reference back into the parser — it owns its data and can outlive the `Options`
object and be returned across scopes[^2]. In version 2 neither was true.

## Production Notes

- **`<regex>` is the hidden cost.** cxxopts includes `<regex>` for numeric
  validation, which is one of the heavier standard headers. In large builds this
  contributes measurable compile time and binary size to every translation unit
  that includes `cxxopts.hpp`. Include it in as few `.cpp` files as possible;
  don't put it in a widely-shared header.
- **Header-only means recompiled everywhere.** There is no prebuilt object; the
  full parser is re-parsed and re-instantiated per TU. For a small tool this is
  invisible; across a large monorepo it is not free.
- **Exceptions are mandatory.** No error-code fallback exists. Codebases built
  with exceptions disabled cannot use cxxopts at all.
- **Boolean-with-value footguns.** `-o false` does not set `o` to false — because
  a bare boolean takes no argument, the `false` is treated as a positional. Use
  `--option=false` / `-o=false`. The same `=`-only rule applies to any option
  with an implicit value: `--option value` won't bind, only `--option=value`.
- **Vector delimiter is compile-time.** List values split on `CXXOPTS_VECTOR_DELIMITER`
  (comma by default), settable only via a `#define`, not per-option. Values with
  embedded commas must be quoted as a whole (`--list="a,b c,d"`).
- **Unicode help width is opt-in.** Correct column alignment for wide/CJK
  characters in `--help` requires building with `CXXOPTS_USE_UNICODE` and ICU;
  by default help width is computed byte-wise.
- **No subcommands, config files, or env-var loading.** These are out of scope by
  design. If you need them, that is the signal to move to a larger library rather
  than bolt them on.
- **Master is a work in progress.** The README explicitly warns to build against a
  tagged release, not `master`[^3].

## When to Use / When Not

**Use when:**
- You want GNU-style flags parsed correctly with minimal ceremony and one header.
- Your CLI is a flat set of options plus positionals (no nested verbs).
- You already use exceptions and want typed access with readable auto-generated help.
- You want a permissively-licensed, widely-packaged dependency you can vendor.

**Avoid when:**
- You need git-style subcommands, shell completion, or config-file/env loading.
- You compile with exceptions disabled or need an error-code API.
- Compile time and binary size are tightly budgeted and you can't afford `<regex>`.
- You want the richest possible feature set — CLI11 or Boost.Program_options fit better.

## Alternatives

- CLIUtils/CLI11 — use instead when you need subcommands, config files, or environment-variable binding; also header-only, larger, more features.
- boostorg/program_options — use when you are already on Boost and want its option/config integration; heavier and requires linking a compiled library.
- p-ranav/argparse — use for a modern C++17, Python-argparse-flavored API when you don't need cxxopts' broad packaging footprint.
- gflags/gflags — use when you want Google-style flags registered globally across translation units rather than a central schema.
- adishavit/argh — use when you want something even smaller and header-only with no parsing/validation opinions at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2014-10-03 | Project started by Jarryd Beck[^1]. |
| 2.x | 2016 onward | Widely-adopted C++11 line; parser mutated `argv`, `ParseResult` tied to parser. |
| 3.0.0 | 2021 | Breaking changes: `const` `argc`/`argv`, self-contained `ParseResult`, reworked exception hierarchy[^2]. |
| 3.x | 2022–2024 | Maintenance and packaging (vcpkg, Conan, Homebrew) on the v3 line. |

Exact minor-release dates are on the GitHub releases page[^4]; only dates verified
above are stated here.

## References

[^1]: Repository created 2014-10-03 (GitHub API `created_at`), authored by Jarryd Beck. https://github.com/jarro2783/cxxopts
[^2]: "Version 3 breaking changes" — cxxopts README (const inputs, parser-independent `ParseResult`). https://github.com/jarro2783/cxxopts#version-3-breaking-changes
[^3]: "Release versions" note — README warns that `master` is a work in progress and to use a tagged release. https://github.com/jarro2783/cxxopts#release-versions
[^4]: cxxopts releases. https://github.com/jarro2783/cxxopts/releases

## Tags

cpp, c-plus-plus, cli, command-line, argument-parser, option-parser, header-only, positional-arguments, gnu-getopt, mit-license
