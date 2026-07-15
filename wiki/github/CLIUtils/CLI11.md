# CLIUtils/CLI11

> A header-only C++11 command-line parser that binds flags and options directly to your program's variables, with subcommands, validators, and config-file round-tripping.

[GitHub repo](https://github.com/CLIUtils/CLI11) ·
[Official website](https://cliutils.github.io/CLI11/book/) ·
[License: BSD-3-Clause](https://github.com/CLIUtils/CLI11/blob/main/LICENSE)

## Overview

CLI11 is a command-line argument parser for C++11 and later, originally
developed by Henry Schreiner at the University of Cincinnati under NSF award
1414736, where it grew out of the GooFit GPU fitting framework[^1]. Its design
was consciously modeled on Python's `plumbum.cli`: the goal is that reading a
value off the command line should be nearly as cheap to write as declaring a
local variable. You create a `CLI::App`, call `add_option`/`add_flag` to bind
names to existing C++ variables, and after parsing those variables hold the
converted values — no lookup table, no `variant` to unpack at every use site.

The library's defining tradeoff is *convenience via templates*. Because
`add_option` accepts almost any type with a known string conversion (integers,
floats, enums, `std::optional`, tuples, containers, `std::atomic`, custom types
with a string constructor), a huge amount of behavior is resolved at compile
time. That buys a clean call site and direct value access — which matters for
the HPC use cases it was built for — at the cost of long compile times and
template error messages that can be hard to read when a binding doesn't match.

CLI11 ships as separate source files but also as a single amalgamated
`CLI11.hpp` generated for every release, so most projects vendor one header and
move on. It has no dependencies beyond the standard library and targets a
deliberately wide compiler floor (GCC 4.8+, Clang 3.4+, MSVC 2015+), which is
part of why it remains common in scientific and cross-platform codebases.

## Getting Started

Vendor the single header, or pull it via CMake `FetchContent`, Conan, or vcpkg.

```cpp
// main.cpp
#include <CLI/CLI.hpp>
#include <string>

int main(int argc, char** argv) {
    CLI::App app{"App description"};
    argv = app.ensure_utf8(argv);          // Windows: widen argv to UTF-8

    std::string file = "default.txt";
    int count = 1;
    app.add_option("-f,--file", file, "Input file")->check(CLI::ExistingFile);
    app.add_option("-n,--count", count, "Repeat count")->capture_default_str();

    CLI11_PARSE(app, argc, argv);          // expands to parse + app.exit(e)
    // file and count now hold parsed (or default) values
    return 0;
}
```

`CLI11_PARSE` is a macro over a `try { app.parse(...) } catch (const
CLI::ParseError& e) { return app.exit(e); }` block; `-h/--help` and parse errors
short-circuit and return the correct `CLI::ExitCodes` value.

## Architecture / How It Works

The object model is small. A `CLI::App` owns a list of `CLI::Option` objects and
a list of child `App`s (subcommands are just nested `App`s). Each `add_option`
call constructs an `Option` holding the option's names, help text, and a
type-erased callback — a lambda captured at call time that knows how to convert
the incoming string(s) into the bound variable's type and assign them. Parsing
walks `argv`, matches tokens to `Option`s (handling `=`, grouped short flags,
the `--` positional separator, and subcommand dispatch), then runs each matched
option's callback. Results land in your variables; there is no intermediate map
you pay to query later.

Type support is where the template weight lives. The one- and two-parameter
`add_option<T, XC>` overloads let you separate the *stored* type from the
*conversion* type, so you can validate that input parses as, say, `unsigned int`
before assigning to a `double`, or steer a `std::variant` toward a specific
alternative. Validators (`CLI::ExistingFile`, `CLI::Range`, `CLI::IsMember`,
transforming validators like `CLI::CheckedTransformer`) attach to options and
compose with `&`/`|`, running as part of the callback.

Configuration files are first-class and bidirectional: an app can read TOML or
INI (or a custom format) into its options and also serialize current values back
out, which makes "generate a config from the CLI" a built-in rather than a
bolt-on. Subcommands support nesting, option groups, required/exclusive
relationships, callbacks fired on match, and optional *fallthrough* (unmatched
options pass up to the parent). Unicode on Windows is handled by `ensure_utf8`,
which converts the wide `argv` the OS provides into UTF-8 `char*` the rest of
the API expects.

## Production Notes

- **Bound variables must outlive `parse`.** `add_option(name, var)` stores a
  reference to `var`. If you bind a local that goes out of scope before parsing
  (a common refactor mistake when parsing is split into a helper), you get a
  dangling reference and undefined behavior, not a compile error.
- **Compile time is the real cost.** The amalgamated `CLI11.hpp` is large and
  heavily templated; including it in many translation units inflates build
  times noticeably. Confine CLI setup to one TU, or use the split headers and a
  precompiled header for large codebases.
- **Template error messages.** When a bound type has no usable string
  conversion, the diagnostic surfaces deep in CLI11's internals rather than at
  your call site. Prefer the two-parameter `add_option<T, XC>` form to make the
  intended conversion explicit.
- **Not thread-safe during parse.** Build and parse an `App` on one thread;
  it's fine to read results concurrently afterward, but concurrent mutation of
  the same `App` is not supported.
- **No built-in shell completion.** Autocomplete generation is explicitly out of
  scope; if you need bash/zsh completion you generate it yourself. Partial
  *prefix* matching for subcommands exists (`allow_subcommand_prefix_matching`),
  but that is not the same as shell completion.
- **Exceptions drive control flow.** Help and parse errors are thrown as
  `CLI::ParseError`. If you build with exceptions disabled you must use the
  lower-level API and handle exit codes manually; the convenience macro assumes
  exceptions are on.
- **Windows argv needs `ensure_utf8`.** Skip it and non-ASCII arguments/paths
  will be mojibake. It reassigns `argv`, so capture the return value.

## When to Use / When Not

**Use when:**
- You want parsed values to land directly in typed C++ variables with minimal
  ceremony.
- You need subcommands, option groups, validators, or config-file read/write in
  one dependency-free header.
- You must support old compilers (CentOS/RHEL 7-era GCC 4.8) or HPC/CUDA builds.

**Avoid when:**
- Build time is precious and your CLI is trivial — a lighter single-header
  parser adds far less compile overhead.
- You compile with exceptions or RTTI disabled and don't want to work around
  the exception-based flow.
- You need generated shell completion out of the box.

## Alternatives

- p-ranav/argparse — C++17 single-header, argparse-style API; leaner if you can
  require C++17 and want a smaller include.
- jarro2783/cxxopts — Boost.PO-like syntax, single file; simpler feature set but
  depends on `<regex>`, so it won't build on GCC 4.8.
- Taywee/args — single-header, optional-like design with subcommand support; use
  when you prefer explicit result objects over direct variable binding.
- muellan/clipp — composable, header-only, expressive grammar; good when your
  argument structure is unusual, at the cost of a steeper API.
- boost/program_options — reach for it only when you already depend on Boost;
  otherwise the pre-C++11 syntax and separate-compilation build are heavier.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017 | First public release; extracted from GooFit, `plumbum.cli`-inspired API[^1]. |
| 2.0 | 2021 | Major release; API cleanups, expanded type support, `char` option semantics changed[^2]. |
| 2.4 | 2024 | `ensure_utf8`/Unicode handling, formatter and validator additions[^2]. |
| 2.5 | 2025 | Config-file and validator improvements[^2]. |
| 2.6.2 | 2026-02-26 | Latest release as of writing[^3]. |

## References

[^1]: CLI11 README — Background/Introduction, on origins in the GooFit fitting
framework and inspiration from Python's `plumbum.cli`.
https://github.com/CLIUtils/CLI11#background
[^2]: CLI11 changelog — per-release feature and breaking-change notes.
https://github.com/CLIUtils/CLI11/blob/main/CHANGELOG.md
[^3]: CLI11 GitHub Releases — v2.6.2, published 2026-02-26.
https://github.com/CLIUtils/CLI11/releases

## Tags

cpp, c-plus-plus, cli, cli-parser, argument-parser, command-line, header-only, cpp11, no-dependencies, subcommands, config-file
