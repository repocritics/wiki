# p-ranav/argparse

> A single-header C++17 command-line argument parser that mirrors Python's `argparse` API.

[GitHub repo](https://github.com/p-ranav/argparse) ·
[License: MIT](https://github.com/p-ranav/argparse/blob/master/LICENSE)

## Overview

`argparse` is a header-only argument parser for C++17, first published in 2019[^1]. Its design goal is explicit: reproduce the ergonomics of Python's standard-library `argparse` in modern C++ — positional and optional arguments, subcommands, mutually exclusive groups, `nargs`, and help/usage generation — while staying small enough to vendor as one file (`argparse.hpp`). It targets developers who want a familiar, declarative CLI definition without pulling in a build-system dependency or a compiled library.

The defining tradeoff is scope versus size. `argparse` deliberately does less than the larger C++ CLI libraries: there is no built-in config-file parsing, no environment-variable binding, and no TOML/INI integration. In exchange you get a dependency you can drop into a project by copying one header, with an API most people already know from Python. For a large fraction of tools — a build script, a converter, an internal utility — that is the entire requirement.

As of mid-2026 the project has ~3.5k stars and is stable rather than fast-moving: the latest tagged line is the 3.x series, and the last repository push was in early 2025[^2]. That cadence reflects a library that is largely feature-complete for its stated scope, not one that is abandoned — but consumers should not expect frequent releases, and open issues (~80) accumulate faster than they are closed.

## Getting Started

Vendor the single header, or install via a package manager (vcpkg, Conan, Homebrew, or CMake `FetchContent`). No linking step is required — it is header-only.

```cpp
#include <argparse/argparse.hpp>
#include <iostream>

int main(int argc, char *argv[]) {
  argparse::ArgumentParser program("square", "1.0");

  program.add_argument("number")
    .help("integer to square")
    .scan<'i', int>();               // convert argv string -> int

  program.add_argument("--verbose")
    .help("print a full sentence")
    .flag();                         // == default_value(false).implicit_value(true)

  try {
    program.parse_args(argc, argv);
  } catch (const std::exception &err) {
    std::cerr << err.what() << "\n" << program;   // program streams its usage
    return 1;
  }

  int n = program.get<int>("number");
  if (program["--verbose"] == true)
    std::cout << "The square of " << n << " is " << n * n << "\n";
  else
    std::cout << n * n << "\n";
  return 0;
}
```

Requires C++17 (`-std=c++17`). GCC, Clang, and MSVC are all supported.

## Architecture / How It Works

The library is two classes. `ArgumentParser` owns the parser state and the registered arguments; `Argument` is the fluent builder returned by `add_argument()`, on which you chain `.help()`, `.default_value()`, `.scan<>()`, `.nargs()`, `.required()`, `.action()`, and so on. Parsing walks `argv` once, matching tokens against registered positionals and optionals, then stores each parsed value type-erased (internally `std::any`). Retrieval is explicit and typed: `program.get<T>(key)` casts the stored `std::any` back to `T`, and `program.present<T>(key)` returns `std::optional<T>` for arguments that may be absent.

Type conversion is compile-time-selected through `scan<Shape, T>()`, where `Shape` is a character code borrowed from format-string conventions — `'i'` for signed integer, `'u'` unsigned, `'d'` decimal, `'g'` general floating-point, `'x'` hex. This is the mechanism that lets the library parse `-5`, `-3.1e2`, and hex literals into the right C++ type while still distinguishing a negative number from an optional flag. Because the type is a template parameter, a mismatch between `scan<>` and the later `get<T>` is a runtime cast failure, not a compile error — a common footgun.

Higher-level features are layered on the same two classes: subcommands are child `ArgumentParser` instances registered with `add_subparser()`; mutually exclusive groups are a small constraint set checked after parsing; `store_into(var)` binds an argument directly to a caller-owned variable (supported only for a fixed set of types — `bool`, `int`, `double`, `std::string`, and vectors of string/int). The Python lineage is visible throughout: `nargs` accepts fixed counts, ranges, and the `any` / `at_least_one` / `optional` patterns that correspond to Python's `*`, `+`, and `?`.

## Production Notes

- **Errors are exceptions, not exit codes.** `parse_args` throws `std::exception` subclasses on bad input; if you do not wrap it in try/catch, a malformed CLI aborts with an uncaught exception rather than a clean usage message. Every example in the README wraps it — that is load-bearing, not boilerplate.
- **`scan` / `get` type must agree.** The value is stored type-erased. Calling `get<double>` on an argument scanned as `int` throws `std::bad_any_cast` at runtime. There is no compiler check tying the two together.
- **`default_value` type must match `get`.** A frequent bug is `default_value("orange")` (a `const char*`) followed by `get<std::string>`, which throws. Use `default_value(std::string{"orange"})`. The README calls this out explicitly.
- **Header-only means compile cost.** `argparse.hpp` is a single large translation-unit include. Pulling it into many `.cpp` files multiplies parse time; for bigger codebases, confine it to one CLI translation unit.
- **Negative-number handling has edge cases.** The parser distinguishes negative numeric operands from options, but combining that with custom prefix characters or unusual `nargs` patterns is where reported issues cluster. Test your exact argument grammar.
- **Release cadence is slow.** With the last push in early 2025 and issues outpacing closes, treat unmerged bug reports as potentially long-lived. For a leaf tool this is fine; if you need an actively triaged dependency in a long-lived product, weigh CLI11.

## When to Use / When Not

**Use when:**
- You want a familiar `argparse`-style API and a one-file, no-link dependency.
- Your CLI is standard: positionals, flags, options with type conversion, subcommands.
- You are on C++17 or newer and value copy-in-and-go over feature breadth.

**Avoid when:**
- You need config-file or environment-variable binding built in (CLI11 or a dedicated config library fit better).
- You want compile-time-checked argument access rather than runtime `std::any` casts.
- You need a dependency with frequent releases and fast issue turnaround.
- You are stuck on C++11/14 — this library requires C++17.

## Alternatives

- CLIUtils/CLI11 — richer feature set (config files, env vars, validators, subcommand callbacks); use instead when your CLI grows beyond flags and positionals.
- jarro2783/cxxopts — lighter header-only parser with a compact API; use when you want minimal surface and don't need subcommands.
- Taywee/args — another single-header option with a similarly Python-flavored feel; use when you want an alternative small dependency.
- gflags/gflags — Google's global flag-registration model; use in large codebases where flags are declared distributed across translation units.
- docopt/docopt.cpp — derives the parser from a written help/usage string; use when you'd rather maintain the help text as the source of truth.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019 | First public release: single-header C++17 parser, Python-`argparse`-style API[^1]. |
| 2.x | 2021–2022 | Subparsers/subcommands, mutually exclusive groups, expanded `nargs` patterns. |
| 3.0 | 2024 | `store_into` variable binding, `flag()` shorthand, `append()`, help-formatting improvements. |
| 3.2 | 2024–2025 | Current line as of last activity; header remains the sole distributed artifact[^2]. |

## References

[^1]: p-ranav/argparse — repository created 2019-03-30, "Argument Parser for Modern C++". https://github.com/p-ranav/argparse
[^2]: GitHub API metadata for p-ranav/argparse (stars, forks, last push 2025-01-26), retrieved 2026-07. https://api.github.com/repos/p-ranav/argparse

## Tags

cpp, cpp17, cli, argument-parser, command-line, header-only, library, mit-license, cross-platform, parsing
