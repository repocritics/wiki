# Hejsil/zig-clap

> The de facto standard command-line argument parser for Zig — comptime-typed results from a help-message-shaped spec.

[GitHub repo](https://github.com/Hejsil/zig-clap) ·
[API reference](https://Hejsil.github.io/zig-clap) ·
[License: MIT](https://github.com/Hejsil/zig-clap/blob/master/LICENSE)

## Overview

zig-clap is a command-line argument parsing library for Zig, started in March 2018[^1] — old enough to predate Zig 0.3, making it one of the longest-lived third-party Zig libraries. At ~1.6k stars it sits near the top of the Zig package ecosystem, and for most Zig CLI projects it is the default answer to "how do I parse args". Its signature trick: you write your parameter specification *as the help text* — a multiline string like `-n, --number <usize>  An option parameter.` — and `parseParamsComptime` parses that string at compile time into typed `Param` values. The result struct then has one field per parameter, typed according to the value name (`<usize>` → `usize`, `<str>...` → slice of strings), so parse results are checked by the compiler rather than fished out of a string map[^2].

The defining tension is not internal to the library — it is Zig itself. Zig is pre-1.0 and breaks its standard library routinely (allocator APIs, the `std.Io` rework, `std.process` changes). zig-clap absorbs each break, so its release history is effectively a shadow of Zig's: each tagged release targets one tagged Zig compiler, and `master` tracks Zig `master`[^3]. Users pick the tag matching their compiler, and upgrading Zig means upgrading zig-clap and adapting to whatever API churn came with it. The library is deliberately small in scope: no subcommand framework, no shell completions, no config-file or environment-variable fallback — parsing, help, and usage output only.

## Getting Started

```sh
# Tagged Zig release: pin the matching zig-clap tag (see releases page)
zig fetch --save https://github.com/Hejsil/zig-clap/archive/refs/tags/0.12.0.tar.gz
# Zig master: track zig-clap master instead
zig fetch --save git+https://github.com/Hejsil/zig-clap
```

```zig
// build.zig
const clap = b.dependency("clap", .{});
exe.root_module.addImport("clap", clap.module("clap"));
```

```zig
// Current master README example (Zig master std.process.Init API);
// older Zig versions use `std.process.args()` and `gpa` directly.
pub fn main(init: std.process.Init) !void {
    const params = comptime clap.parseParamsComptime(
        \\-h, --help             Display this help and exit.
        \\-n, --number <usize>   An option parameter, which takes a value.
        \\<str>...
        \\
    );

    var diag = clap.Diagnostic{};
    var res = clap.parse(clap.Help, &params, clap.parsers.default, init.minimal.args, .{
        .diagnostic = &diag,
        .allocator = init.gpa,
    }) catch |err| {
        try diag.reportToFile(init.io, .stderr(), err);
        return err;
    };
    defer res.deinit();

    if (res.args.help != 0)
        return clap.helpToFile(init.io, .stderr(), clap.Help, &params, .{});
    if (res.args.number) |n|
        std.debug.print("--number = {}\n", .{n});
}

const clap = @import("clap");
const std = @import("std");
```

## Architecture / How It Works

The library is layered, smallest primitive at the bottom[^2]:

- **`streaming.Clap`** — the base parser. Pulls arguments lazily from an iterator and yields one matched `(param, value)` pair per `next()` call. It is the only layer that accepts a runtime-constructed `[]Param`; every higher layer requires the spec at comptime.
- **`clap.parse` / `clap.parseEx`** — drive the streaming parser to completion and materialize a typed result. `parse` reads process args directly; `parseEx` takes any iterator, which is what makes subcommands possible: set `.terminating_positional` to stop after the subcommand name, then hand the half-consumed iterator to a second `parseEx` call with the subcommand's own params.
- **`clap.parsers`** — the string-to-type mapping. The value name in the spec (`<usize>`, `<str>`) is looked up as a field in the parsers struct; the parser function's return type becomes the result field's type. Supplying your own struct (e.g. `clap.parsers.enumeration(MyEnum)`) is how custom types and enums enter the type system.
- **`clap.help` / `clap.usage`** — generate the help text and the abbreviated usage line back from the same `Param` array, closing the loop: the spec string is both documentation and schema.

Field naming follows the longest name of each parameter; flags count occurrences (`-v -v` → `2`), optional values are optionals, repeated options are slices. Error reporting is opt-in via `Diagnostic`; without it, a failed parse surfaces only a bare Zig error code.

Argument syntax support is thorough for GNU-style conventions: short chaining (`-abc`), values with space, `=`, or fused (`-a100`), configurable assignment separators, and multi-occurrence options.

## Production Notes

- **Version lockstep is the main operational cost.** A Zig compiler upgrade almost always requires a zig-clap upgrade, and the API surface has churned with Zig's std: e.g. the `std.Io` rework turned `diag.report(...)` into `diag.reportToFile(io, ...)` and `clap.help` into `clap.helpToFile`. README snippets on `master` will not compile on tagged Zig releases — match docs to your tag.
- **Comptime spec means no dynamic CLIs.** If parameters depend on runtime data (plugins, config-driven flags), only the lower-level `streaming.Clap` accepts a runtime `[]Param`, and you lose typed results and must dispatch on ids yourself.
- **Memory:** `clap.parse` allocates (multi-value options accumulate into slices); `res.deinit()` is mandatory or you leak under a GPA in debug.
- **Scope gaps you will fill yourself:** subcommand dispatch is a primitive (`terminating_positional`), not a framework — no automatic per-subcommand help routing, no shell completion generation, no derived version flag.
- **Bus factor:** effectively a single-maintainer project (Hejsil). Activity is steady but release cadence follows Zig's, so fixes for a new compiler can lag a few weeks after a Zig release. The codebase is small enough to vendor and patch if stuck. 7 open issues as of mid-2026 suggests triage is keeping up.
- The generated API reference site uses Zig autodoc, which the README itself flags as beta and possibly broken[^2].

## When to Use / When Not

**Use when:**
- You are writing a Zig CLI and want typed, compiler-checked argument access with near-zero dependency weight.
- You want help/usage output derived from a single source of truth.
- Your flag set is fixed at compile time (the overwhelmingly common case).

**Avoid when:**
- You need a batteries-included CLI framework: nested subcommand trees with auto-help, completions, env/config integration.
- Your parameter set is constructed at runtime and you want typed results.
- You are not on Zig — this library is idiomatically and technically inseparable from the Zig compiler version you build with.

## Alternatives

- prajwalch/yazap — use instead when you want a builder-style API with first-class nested subcommands rather than a comptime spec string.
- MasterQ32/zig-args — use instead when you prefer declaring options as a plain Zig struct and letting field names define the flags.
- sam701/zig-cli — use instead when you want an app/command framework (commands, actions, help wiring) rather than just a parser.
- 00JCIV00/cova — use instead when you need completions and richer command/option metadata and accept a heavier framework.
- clap-rs/clap — the Rust namesake; use instead when the tool does not need to be Zig at all and you want a mature, feature-complete ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2019-01-17 | First tagged release; repo created 2018-03[^1]. |
| v0.3.0 | 2020-11-10 | Early comptime-spec era. |
| 0.5.0 | 2021-12-21 | Pre-package-manager; consumed via git submodule/vendor. |
| 0.6.0 | 2022-11-01 | Aligned with the Zig 0.10 cycle[^3]. |
| 0.7.0 | 2023-08-18 | Zig package manager (`zig fetch` / `build.zig.zon`) era begins. |
| 0.8.0 | 2024-04-21 | Tracks Zig 0.12 (released the previous day)[^3]. |
| 0.9.0 | 2024-06-19 | Zig 0.13 cycle. |
| 0.10.0 | 2025-03-05 | Published the same day as Zig 0.14.0[^3]. |
| 0.11.0 | 2025-08-21 | Zig 0.15 cycle (`std.Io` rework fallout). |
| 0.12.0 | 2026-04-20 | Current tagged release[^4]. |

## References

[^1]: GitHub repository metadata — created 2018-03-14; ~1.6k stars, 99 forks, MIT, last push 2026-06-13 (as of 2026-07-18). https://github.com/Hejsil/zig-clap
[^2]: zig-clap README — installation, `parseParamsComptime`, layered parsers, streaming API, autodoc caveat. https://github.com/Hejsil/zig-clap#readme
[^3]: Zig release history (0.10.0 2022-10, 0.12.0 2024-04-20, 0.14.0 2025-03-05), against which zig-clap tags align. https://ziglang.org/download/
[^4]: zig-clap releases. https://github.com/Hejsil/zig-clap/releases

## Tags

zig, argument-parser, cli, command-line, comptime, parsing, library, zig-package
