# kubkon/zig-yaml

> A pure-Zig YAML parser that maps documents onto Zig structs via comptime reflection — honest about being a partial YAML 1.2 implementation, not a spec-complete one.

[GitHub repo](https://github.com/kubkon/zig-yaml) ·
[License: MIT](https://github.com/kubkon/zig-yaml/blob/main/LICENSE)

## Overview

zig-yaml is a YAML parser written in Zig with no C dependencies, started in 2021 by Jakub Konka (kubkon), a Zig core team member best known for the self-hosted linker work[^1]. It targets YAML 1.2 "one step at a time" — the README says plainly that it is a work-in-progress and that breakage should be expected[^2]. That candor is accurate: the library handles the common subset (mappings, sequences, explicit documents, flow and block styles) and tracks its YAML spec-suite failures in a dedicated meta-issue rather than claiming conformance[^3].

Its defining trade-off is scope versus purity. Wrapping libyaml gets you a battle-tested YAML 1.1 implementation but drags C into your build graph; zig-yaml gets you idiomatic Zig — typed decoding into user structs via comptime reflection, arena-based memory management, error reporting through `std.zig.ErrorBundle` — at the cost of spec coverage. For config files and tooling formats that use plain YAML, that subset is usually enough; for consuming arbitrary third-party YAML, it may not be.

At ~295 stars and 87 forks it is small but is the default answer to "YAML in Zig", partly because of the author's standing in the ecosystem. Maintenance is real but low-cadence: releases cluster around Zig compiler version bumps (0.1.x for Zig 0.14 in March 2025, 0.2.0 in January 2026), and the last push as of mid-2026 was the 0.2.0 release itself[^4].

## Getting Started

```bash
# fetch a tagged release into build.zig.zon (name it "yaml" for convenience)
zig fetch --save=yaml https://github.com/kubkon/zig-yaml/archive/refs/tags/0.2.0.tar.gz
```

```zig
// build.zig — after b.installArtifact(exe)
const yaml = b.dependency("yaml", .{ .target = target, .optimize = optimize });
exe.root_module.addImport("yaml", yaml.module("yaml"));
```

```zig
const std = @import("std");
const Yaml = @import("yaml").Yaml;

const Config = struct {
    name: []const u8,
    ports: []const u16,
};

pub fn main() !void {
    const gpa = std.heap.page_allocator;
    var yaml: Yaml = .{ .source = "name: server\nports: [8080, 8443]" };
    defer yaml.deinit(gpa);
    try yaml.load(gpa);

    var arena = std.heap.ArenaAllocator.init(gpa);
    defer arena.deinit();
    const cfg = try yaml.parse(arena.allocator(), Config);
    std.debug.print("{s}: {d} ports\n", .{ cfg.name, cfg.ports.len });
}
```

Version pinning matters: `0.0.1` is for pre-0.14 Zig (e.g. 0.13), `0.1.x` for Zig 0.14, `0.2.0` for later compilers[^2].

## Architecture / How It Works

The pipeline is conventional: a tokenizer feeds a parser that builds a document tree, which is then lowered into an untyped value representation (maps, lists, scalars). Three entry points sit on top:

1. **`Yaml.load`** — untyped access. You get `docs` (YAML supports multiple documents per stream) and walk maps/lists manually.
2. **`Yaml.parse(allocator, T)`** — typed decoding. Uses Zig's comptime reflection (`@typeInfo`) to map YAML keys onto struct fields, sequences onto slices and arrays, and scalars onto ints/floats/strings. This is the library's main selling point: no codegen step, no runtime schema — the struct *is* the schema. The intended pattern is to pass an `ArenaAllocator` and free everything at once; per-field ownership is not tracked.
3. **`Yaml.stringify`** — serializes a `Yaml` value back to text. Output is normalized (flow style for lists), not round-trip-faithful to the input's formatting.

Parse errors are stored out-of-band in `Yaml.parse_errors` as a `std.zig.ErrorBundle` — the same machinery the Zig compiler uses for diagnostics — so callers render them with source locations and TTY colors rather than matching on a bare error enum[^2]. The repo vendors the official YAML test suite as a submodule behind `zig build test -Denable-spec-tests`, with known-failing cases explicitly skip-listed in `test/spec.zig`[^3].

The project's origin shapes its priorities: it came out of the Zig macOS linker effort, where Apple's `.tbd` text-based dylib stubs (a YAML dialect) needed parsing, and a derivative of it lives inside the Zig compiler's Mach-O linker code. Correctness on tooling-style YAML was the design center, not full spec conformance.

## Production Notes

- **Spec gaps are the main risk.** Anchors/aliases, complex tag handling, and assorted edge cases from the YAML test suite fail or are skipped[^3]. If you accept YAML from users or third parties, test your actual corpus against it before committing — do not assume "it's YAML, it parses".
- **Zig version lock-in.** Zig has no stable language yet; every compiler release breaks libraries. zig-yaml's releases map to compiler windows, so upgrading Zig usually means waiting for (or patching) a matching zig-yaml tag. Between January 2025 and January 2026 there were four releases, driven substantially by upstream churn[^4].
- **Memory model requires attention.** Typed `parse` results borrow from an arena you supply. Free the arena, not individual fields, and keep the `Yaml` struct alive as long as you use untyped values.
- **33 open issues against a small surface** (as of mid-2026) signal known rough edges, not abandonment — this is a side project of a compiler engineer, not a staffed team[^4].
- **Stringify is lossy** with respect to formatting and comments. Do not use it for editing YAML files that humans maintain.

## When to Use / When Not

**Use when:**
- You need YAML config parsing in a pure-Zig build with no C dependencies.
- Your YAML is under your control (CI configs, manifests, tool metadata).
- You want typed decoding into structs without a schema DSL or codegen.

**Avoid when:**
- You must parse arbitrary externally-supplied YAML (anchors, merge keys, exotic tags) — wrap libyaml or preprocess with a conforming parser.
- You control the format: Zig's built-in ZON or `std.json` give you first-party parsers with no third-party dependency.
- You need round-trip editing that preserves comments and formatting.

## Alternatives

- yaml/libyaml — use instead when you need mature, spec-hardened YAML parsing and can tolerate a C dependency via Zig's C interop.
- ziglang/zig — `std.json` (and ZON for build-adjacent data) instead when you control the format; zero external dependencies.
- sam701/zig-toml — use instead when config is the goal and TOML's smaller, saner spec is acceptable to your users.

## History

| Version | Date | Notes |
|---------|------|-------|
| (repo created) | 2021-05-31 | Started pre-package-manager; consumed via git for years[^4]. |
| 0.0.1 | 2025-01-17 | First tagged release; targets pre-0.14 Zig (e.g. 0.13)[^4]. |
| 0.1.0 | 2025-03-07 | Zig 0.14 support[^4]. |
| 0.1.1 | 2025-03-11 | Patch follow-up to 0.1.0[^4]. |
| 0.2.0 | 2026-01-13 | Current release; tracks newer Zig compilers[^4]. |

## References

[^1]: Jakub Konka — GitHub profile; Zig core team, linker work. https://github.com/kubkon
[^2]: zig-yaml README. https://github.com/kubkon/zig-yaml#readme
[^3]: Meta-issue tracking failing YAML spec-suite tests. https://github.com/kubkon/zig-yaml/issues/48
[^4]: GitHub repo metadata and releases, fetched 2026-07-18. https://github.com/kubkon/zig-yaml/releases

## Tags

zig, yaml, parser, serialization, config, zig-library, comptime, no-dependencies
