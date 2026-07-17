# Taywee/args

> A single-header C++11 command-line argument parser modeled on Python's argparse, with static type checking and fully nestable argument groups.

[GitHub repo](https://github.com/Taywee/args) ·
[Official website](https://taywee.github.io/args/) ·
[License: MIT](https://github.com/Taywee/args/blob/master/LICENSE)

## Overview

args is a header-only C++11 library (`args.hxx`, a single file) for parsing command-line arguments. It takes its conceptual model from Python's `argparse` — parsers, flags, value flags, positionals, and lists — but reimplements it in C++ with compile-time type checking, so a `ValueFlag<int>` yields an `int` and a malformed value raises a parse error rather than being silently coerced[^1]. It deliberately does not mirror argparse's Python API.

The maintainer declares the library feature-complete: it still accepts bug fixes and good pull requests, but no new major functionality or API changes are planned[^1]. For a CLI parser this is a defensible end state rather than abandonment — the argument-parsing problem is well-bounded, and a stable single header is exactly what many projects want to vendor. The tradeoff is that anything the library does not already do (see below) will not be added upstream, so needs beyond its scope mean patching locally or picking a different library.

Its distinguishing feature over lighter parsers is **nestable argument groups** with validators (`Xor`, `And`, `AtLeastOne`, `AtMostOne`, `DontCare`, or a user-supplied function). You can express constraints like "exactly one of `-f`/`-b`/`--baz`" declaratively. The cost of that generality shows up in error reporting, which the library itself calls out as its main weakness.

## Getting Started

There is no build step for the library — it is one header. Copy `args.hxx` into your source tree, or install it system-wide:

```shell
sudo make install                 # installs to /usr/local by default
```

It is also packaged for CMake, Conan, Meson, Buck, and vcpkg. A minimal program:

```cpp
#include <iostream>
#include <args.hxx>

int main(int argc, char **argv)
{
    args::ArgumentParser parser("This is a test program.");
    args::HelpFlag help(parser, "help", "Display this help menu", {'h', "help"});
    args::ValueFlag<int> count(parser, "count", "The count", {'c', "count"});
    args::Positional<std::string> name(parser, "name", "A name to greet");

    try
    {
        parser.ParseCLI(argc, argv);
    }
    catch (const args::Help&)
    {
        std::cout << parser;
        return 0;
    }
    catch (const args::ParseError& e)
    {
        std::cerr << e.what() << std::endl << parser;
        return 1;
    }

    if (name)  { std::cout << "Hello, " << args::get(name) << std::endl; }
    if (count) { std::cout << "count: " << args::get(count) << std::endl; }
    return 0;
}
```

Values are retrieved with `args::get(...)`; the argument objects themselves convert to `bool` to report whether they matched.

## Architecture / How It Works

Everything lives in the `args` namespace inside one header. The building blocks are objects that register themselves with a parent parser or group at construction:

- **`ArgumentParser`** — the root. `ParseCLI(argc, argv)` consumes the program name plus arguments; `ParseArgs(...)` consumes just the arguments.
- **`Flag`** — boolean presence. **`ValueFlag<T>`** — takes one value parsed to `T`. **`ValueFlagList<T>`** — repeatable, collects a container. **`Positional<T>`** / **`PositionalList<T>`** — positional equivalents. **`HelpFlag`**, **`CompletionFlag`**, **`CounterFlag`** — special-purpose flags.
- **`Matcher`** — the set of short (`'h'`) and long (`"help"`) option names bound to a flag, constructed inline as `{'h', "help"}`.
- **`Group`** — a container whose child arguments participate in a validator. Groups nest arbitrarily, so a validator expression can be an arbitrary boolean tree over sub-groups[^1].

**Type parsing** defaults to `operator>>` (stream extraction): any type with a stream extractor can be a `ValueFlag<T>` for free. If that is wrong for your type, you pass a reader functor as a second template parameter and parse the string yourself. The built-in validator only errors when characters are left unextracted in the stream, which means partial/loose parses can slip through unless your reader is strict[^1].

**Subcommands** are modeled with `Command` and `Subparser`. A `Command` can carry a coroutine-style callback (a function or lambda taking `args::Subparser&`) so each subcommand's flags and logic live together, and `GlobalOptions` / `args::Options::Global` share flags across all subcommands (the git-`--git-dir` pattern).

**Error handling is exception-based.** `args::Help` and `args::Completion` signal normal early exits (print help / print completion); `args::ParseError` and `args::ValidationError` signal user error; all derive from `args::Error`. There is no error-code return path — you must wrap `ParseCLI` in try/catch.

## Production Notes

**Group-validation errors are intentionally unhelpful.** The README is explicit: because a group validator can be any boolean expression like `(A && B) || (C && (D XOR E))`, the library only knows the whole expression evaluated false, not which term the user got wrong. Failed validation prints a generic "Group validation failed somewhere!" To give users actionable messages you must catch `ValidationError` and re-derive the failure by inspecting your own group state[^1]. Budget for this if your CLI has non-trivial constraints.

**Ordering constraints.** args will not make flags order-sensitive relative to positionals, and a positional *list* slurps all following positionals — so you cannot place a `PositionalList` before other positionals and expect them to be filled. This is a design decision made for parsing speed and static checking, not a bug, and the library does not stop you from constructing the invalid arrangement[^1].

**UTF-8 handling is limited.** No normalization is performed; non-ASCII characters in flag names and combined glyphs can corrupt help-output alignment. Keep flags ASCII[^1].

**Custom-type footguns.** Because default parsing leans on `operator>>` and only checks for leftover stream characters, custom readers that call `std::stod`/`std::stoi` can throw `std::invalid_argument` or `std::out_of_range` that escape as uncaught exceptions (the README demonstrates this aborting the program). Validate and throw an `args::ParseError` yourself inside custom readers[^1].

**Compile-time cost.** As a template-heavy single header included in every translation unit that parses args, it adds to compile time in the including file, though for a CLI entry point this is usually negligible. There is no runtime dependency and nothing to link.

**Stability.** With upstream feature-frozen, the practical upgrade risk is low: pinning a copy of `args.hxx` in-tree gives a fixed API for the life of the project, which is a common and supported usage per the README[^1].

## When to Use / When Not

**Use when:**
- You want a dependency-free, vendor-in-one-file parser for a C++11 (or newer) project.
- You need declarative, nestable group constraints (mutually exclusive / required-together option sets).
- You want static typing on parsed values and are comfortable with exceptions.
- You are building git-style subcommands and want their flags scoped per command.

**Avoid when:**
- You need polished, automatic error messages for complex constraint failures out of the box (CLI11 does better here).
- Your project is C++17+ and you want an API that tracks Python's argparse more closely (p-ranav/argparse).
- You need config-file or environment-variable binding (Boost.Program_options, gflags).
- You require robust internationalized/UTF-8 flag handling.

## Alternatives

- CLIUtils/CLI11 — richer single-header parser with config files, better validation and constraint error messages; use it when you need friendlier failure output and don't mind more code.
- p-ranav/argparse — C++17 header-only, closest to Python argparse ergonomics; use it when you're on C++17 and want the argparse API feel.
- jarro2783/cxxopts — smaller single-header parser with a simpler model; use it when you want minimal surface and no nested groups.
- Boost.Program_options — heavyweight, supports positional + config-file + env sources; use it when you already depend on Boost and need multi-source options.
- gflags/gflags — Google's distributed global-variable flags; use it in large codebases where flags are declared across many files.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2016-04-28 | Repo created; single-header argparse-style C++11 parser[^2]. |
| 6.x series | current | `Command`/`Subparser` subcommands, `CompletionFlag` bash completion, `GlobalOptions`, nestable group validators[^1]. |
| feature-complete | ongoing | Maintainer freezes major functionality; bug fixes and PRs still accepted[^1]. |

## References

[^1]: Taywee/args README — features, installation, group-validation caveat, UTF-8 note, feature-complete declaration, and usage examples. https://github.com/Taywee/args/blob/master/README.md
[^2]: GitHub repository metadata (created 2016-04-28, MIT license, C++). https://github.com/Taywee/args

## Tags

cpp, cpp11, header-only, argument-parser, command-line, cli, argparse, single-header, library, parsing
