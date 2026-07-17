# adishavit/argh

> A single-header C++11 command-line parser that deliberately does no validation, throws no exceptions, and generates no usage message.

[GitHub repo](https://github.com/adishavit/argh) ·
[License: BSD-3-Clause](https://github.com/adishavit/argh/blob/master/LICENSE)

## Overview

`argh` is a header-only argument parser for C++11 and later, first published by Adi Shavit in 2016[^1]. Its thesis is a reaction against the two dominant styles of C++ CLI parsing: heavyweight frameworks that drag in dependencies (the README singles out `Boost.Program_options`) and verbose builder-DSLs that take over `main()` in exchange for auto-generated help text and validation. `argh` declines both. It parses `argv` into three buckets — positional arguments, boolean flags, and named parameters — and hands them back through `operator[]` and `operator()`. Everything else is the caller's problem.

The defining tradeoff is *laissez-faire by design*. `argh` does not care how many dashes precede an option, does not know which flags your program supports, does not validate syntax, and does not produce a `--help` message[^2]. Type conversion is deferred to the call site via `std::istream` extraction, so a bad `--threshold=abc` surfaces as a failed stream, not a thrown exception — you check it where you use it. This makes `argh` well suited to prototypes, internal tools, and code that wants argument access without ceremony, and a poor fit for polished user-facing CLIs that need help output, subcommands, or maintainer-enforced validation.

At ~1.4k stars the project is small but long-lived. It is not archived, but development has slowed markedly: the last tagged release, v1.3.2, is from 2022, and commits since are sparse maintenance rather than feature work. For a single-header file with a stable, minimal API, low churn is closer to "done" than "abandoned" — but expect to own any behavior it does not give you.

## Getting Started

There is nothing to build or link. Copy the single `argh.h` into your include path, add it as a submodule, or consume it via CMake (`find_package(argh)` exports an `argh` INTERFACE target).

```cpp
#include <iostream>
#include "argh.h"

int main(int, char* argv[])
{
    argh::parser cmdl(argv);          // argc is optional

    if (cmdl[{ "-v", "--verbose" }])  // boolean flag, any dash count
        std::cout << "verbose on\n";

    float scale = 1.0f;               // default if absent or unconvertible
    cmdl("scale", 1.0f) >> scale;     // parameter via istream extraction

    std::string input;
    if (cmdl(1) >> input)             // 1st positional arg after exe name
        std::cout << "input: " << input << '\n';
}
```

`--scale=2.5` binds by default; the spaced form `--scale 2.5` only works if `scale` is pre-registered as a parameter or `PREFER_PARAM_FOR_UNREG_OPTION` mode is set (see below).

## Architecture / How It Works

The whole library is one class, `argh::parser`, backed by three containers: an ordered vector of positional strings, a set of flag names, and a multimap of parameter name→value pairs. Parsing is a single linear pass over `argv`. There is no grammar, no AST, and no registration requirement for most usage.

The central ambiguity every CLI parser must resolve is: given `--foo bar`, is `bar` the value of parameter `foo`, or is `foo` a flag and `bar` a separate positional? `argh` cannot know your intent, so it exposes the decision as a parse *mode*[^2]:

- **`PREFER_FLAG_FOR_UNREG_OPTION`** (default) — unregistered `--foo` is a flag;
  `bar` becomes positional. Predictable, but means `--foo bar` parameters must
  be pre-registered.
- **`PREFER_PARAM_FOR_UNREG_OPTION`** — any `<option> <non-option>` pair is read
  as parameter→value. Convenient, but swallows what you may have meant as a
  standalone flag followed by a positional.
- **`NO_SPLIT_ON_EQUALSIGN`** — disables the default `--foo=bar` splitting.
- **`SINGLE_DASH_IS_MULTIFLAG`** — POSIX-style compound flags, so `-xvf` becomes
  three flags `x`, `v`, `f` (with the last character optionally a param).

Pre-registration (`add_param`, `add_params`, or the initializer-list ctor) exists to disambiguate the default mode: it tells the parser that a given name takes a following value. This is the only "schema" `argh` has, and it is optional.

Access is via two operator families. `operator[]` returns booleans and raw strings: `cmdl["v"]` (flag present?), `cmdl[0]` (positional by index, empty string if out of bounds). `operator()` returns a `std::istream`: `cmdl("scale") >> x` extracts and typed-converts, and standard stream state signals both "argument missing" and "conversion failed" identically. Duplicate parameters are preserved and iterable via `params("name")`; `pos_args()`, `flags()`, and `params()` expose the containers directly for range-`for`.

The design consequence worth internalizing: **"missing" and "invalid" collapse into one failure signal.** If you need to distinguish "user omitted `--port`" from "user passed `--port=xyz`", you must check presence with `[]`/`()` before extracting. Because conversion is `std::istream >>`, values with embedded whitespace (paths, quoted strings) get tokenized unless you use `.str()` instead of `>>`.

## Production Notes

- **No validation is a feature and a footgun.** `argh` will happily accept
  typos, unknown flags, and malformed options without complaint —
  `cmdl["--verbse"]` is simply `false`. There is no "unknown option" error
  because the parser has no notion of a known option set. Programs that need to
  reject bad input must build that layer themselves.
- **No usage/help generation.** You write and maintain `--help` text by hand,
  and keep it in sync with the flags you actually read. For user-facing tools
  this hand-sync burden is exactly what heavier libraries automate away.
- **`istream` tokenization on values.** `cmdl({"-c","--config"}) >> path` stops
  at the first whitespace. For values that can contain spaces, use
  `.str()`: `auto path = cmdl({"-c","--config"}).str();`. This bites people
  passing Windows paths or quoted config strings.
- **Number-vs-option heuristic.** Options are args beginning with `-` that are
  *not* negative numbers, so `-3.14` is treated as a positional value, not an
  option. This is usually what you want but is worth knowing when your program
  legitimately uses short flags that look numeric.
- **Mode is a global parse decision, not per-option.** You pick one
  flag/param disambiguation policy for the whole command line; you cannot say
  "treat `--a` as param but `--b` as flag" except through pre-registration.
- **Maintenance cadence.** With the last release in 2022 and low commit
  activity, do not expect new features. The API surface is small and stable
  enough that this is tolerable, but you are effectively adopting a frozen
  design. Vendor the header and move on rather than tracking upstream.

## When to Use / When Not

**Use when:**
- You want zero-dependency, header-only argument access in a prototype,
  internal tool, or example program.
- You are comfortable owning validation and help text yourself.
- You value a tiny, readable, single-file dependency you can vendor and forget.
- You want typed extraction via familiar `std::istream` semantics.

**Avoid when:**
- You are shipping a polished user-facing CLI that needs auto-generated help,
  subcommands, required-argument enforcement, or shell completion.
- You want the parser to reject unknown or malformed options for you.
- Your team expects a schema/spec as the single source of truth for arguments.
- You need mutually-exclusive groups, dependent options, or rich error messages
  out of the box.

## Alternatives

- CLIUtils/CLI11 — header-only C++11 parser with validation, subcommands, and
  auto help; use it when you want structure and error messages, not minimalism.
- jarro2783/cxxopts — lightweight, getopt-like single-header parser; use when
  you want a middle ground with typed values and generated help.
- p-ranav/argparse — Python `argparse`-style API for modern C++; use when you
  want that ergonomics and don't mind a larger surface.
- boostorg/program_options — the heavyweight it reacts against; use when you
  need config-file + env-var + CLI unification and already depend on Boost.
- muellan/clipp — expressive DSL with man-page generation; use when you want
  rich declarative command specs and are willing to pay in verbosity.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-08-05 | First public commit of the single-header parser[^1]. |
| v1.1.0 | 2017-09-09 | Early tagged release. |
| v1.2.0 | 2017-09-10 | Parameter/flag handling refinements. |
| v1.2.1 | 2018-03-06 | Maintenance release. |
| v1.3.0 | 2018-11-26 | API additions around parameter access. |
| v1.3.1 | 2019-04-02 | Bug fixes; predates the CppCon 2019 talk. |
| v1.3.2 | 2022-03-28 | Latest tagged release as of 2026[^3]. |

The author presented the library's philosophy in "Arguments over Arguments" at
Core C++ 2019 and CppCon 2019[^1].

## References

[^1]: adishavit/argh README and repository. https://github.com/adishavit/argh
[^2]: argh README — Philosophy, Special Parsing Modes, and Argument Access sections. https://github.com/adishavit/argh#readme
[^3]: argh releases — v1.3.2 published 2022-03-28. https://github.com/adishavit/argh/releases

## Tags

cpp, cpp11, command-line-parser, argument-parser, header-only, single-file, cli, getopt, minimalist, no-dependencies
