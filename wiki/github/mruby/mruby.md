# mruby/mruby

> The lightweight, embeddable Ruby — Matz's own answer to "why is Lua in your app instead of Ruby?"

[GitHub repo](https://github.com/mruby/mruby) ·
[Official website](https://mruby.org) ·
[License: MIT](https://github.com/mruby/mruby/blob/master/LICENSE)

## Overview

mruby is a compact implementation of Ruby designed to be linked into and embedded within other programs, led by Ruby's creator Yukihiro "Matz" Matsumoto. It first appeared in 2012, funded by a Japanese Ministry of Economy, Trade and Industry R&D program[^1], and implements (part of) the ISO/IEC 30170 Ruby standard[^2] while tracking modern syntax — the current line advertises Ruby 4.x syntax compatibility[^1]. Where CRuby is a batteries-included application runtime, mruby is a library: a `mrb_state` you open, feed code, and close, the way you would embed Lua.

The defining tradeoff is compile-time configuration versus runtime dynamism. mruby has no `require`, no RubyGems, and no runtime package loading; functionality is selected at build time through **mrbgems**, and everything outside a deliberately small core (including `Regexp`, `IO`, and much of what Rubyists consider "just Ruby") is a gem you compile in or live without[^3]. The result can run in hundreds of kilobytes on microcontrollers, inside web servers (ngx_mruby, h2o), and as a sandboxed scripting layer — Shopify embedded it for merchant scripting via mruby-engine — but a CRuby program will generally not run unmodified.

At 5,590 stars it is a niche project relative to its pedigree, yet unusually well-tended: the repo shows commits within days (last push 2026-07-16) and essentially zero open issues, because the tracker doubles as the project's support channel and is triaged aggressively — Matz himself remains a daily committer.

## Getting Started

Building requires a host CRuby and Rake (an ironic bootstrap dependency):

```bash
git clone https://github.com/mruby/mruby.git
cd mruby
rake all test          # builds bin/mruby, bin/mirb, bin/mrbc + libmruby.a
```

Embedding the VM in C:

```c
#include <mruby.h>
#include <mruby/compile.h>

int main(void) {
  mrb_state *mrb = mrb_open();
  if (!mrb) return 1;
  mrb_load_string(mrb, "3.times { puts 'hello from mruby' }");
  mrb_close(mrb);
  return 0;
}
```

```bash
gcc -I build/host/include app.c build/host/lib/libmruby.a -lm -o app
```

Alternatively, `rake amalgam` produces a single `mruby.c`/`mruby.h` pair (SQLite-style) for drop-in embedding[^1].

## Architecture / How It Works

mruby is a classic three-part interpreter, all reachable through a C API rooted in `mrb_state`:

- **Compiler** — `mrbc` (or `mrb_load_string` at runtime) parses Ruby source and emits RITE bytecode. Bytecode can be dumped as a C array, so firmware ships precompiled scripts with no parser on the device.
- **VM** — a register-based virtual machine executing byte-oriented instructions[^4]. mruby 2.0 replaced the original fixed 32-bit word opcodes with this encoding, one of several deliberate breaks in the project's history.
- **GC** — an incremental tri-color mark-and-sweep collector with a C-visible protection mechanism called the **GC arena**: every object a C function creates is pinned in the arena until you call `mrb_gc_arena_restore`. Forgetting this in a loop is the canonical mruby C-extension bug — the arena grows until memory is exhausted — and the project ships a dedicated guide for it[^5].

Value representation is configurable at build time (word boxing by default, NaN boxing optional), as are integer width and `Float` precision (float32 for constrained targets). The allocator is pluggable via `mrb_open_allocf`, and ROM method tables let class method tables live in flash rather than RAM on microcontrollers[^6].

**mrbgems** is the coupling story: the build is driven by a Ruby `build_config` file that lists gems and cross-compilation targets; "gemboxes" like `default.gembox` and `full-core.gembox` bundle common sets. Gems can mix C and Ruby source and are compiled into `libmruby.a`. There is no dynamic loading path in core — the binary you link is the language you get.

## Production Notes

- **It is not CRuby.** `doc/limitations.md` is required reading[^3]. No `Regexp` in core (add `mruby-onig-regexp` or similar, and note string APIs that take regexes differ until you do), no threads, no `require`, and stdlib coverage is a fraction of CRuby's. Budget porting time for any existing Ruby code; treat mruby as "Ruby-syntax scripting" rather than "Ruby".
- **Bytecode is not a stable ABI.** RITE binary formats change between versions; `.mrb` files must be recompiled with a matching `mrbc`. Do not ship precompiled bytecode from one mruby version into a host embedding another.
- **`mrb_state` is not thread-safe.** There is no GVL because there are no threads; concurrency means one interpreter per thread (interpreters are cheap) or Fibers via `mruby-fiber` within one.
- **The GC arena will bite your C extensions.** Wrap object-allocating loops in `mrb_gc_arena_save`/`restore`[^5]. Symptoms are slow memory growth or, with debug builds, arena overflow aborts.
- **Version upgrades are real work.** 2.0 broke bytecode and parts of the C API; 3.0 reworked symbol handling (preallocated symbols) to shrink binaries[^7]. The project favors correctness and size over compatibility guarantees, on a roughly annual release cadence since 3.0.
- **Third-party gem quality varies widely.** The mgem ecosystem is long-tailed and many entries are unmaintained; vet each gem's CI status against your target mruby version before depending on it.
- **Numeric behavior is build-dependent.** float32 builds and configurable integer width mean arithmetic can differ across targets of the same codebase — a surprise if scripts move between MCU and host builds.

## When to Use / When Not

**Use when:**
- You are embedding a scripting language in a C/C++ application and your team prefers Ruby ergonomics over Lua.
- You need Ruby on microcontrollers or other memory-constrained targets (hundreds of KB, no OS assumed).
- You want sandboxed user scripting with a fixed, auditable feature surface — no `require` is a security feature here.
- You ship firmware and want precompiled bytecode with no parser on-device.

**Avoid when:**
- You expect to run existing CRuby code or gems — compatibility gaps make this a porting project, not a drop-in.
- You need a rich stdlib, native threads, or runtime code loading; use CRuby.
- Binary size and embedding API maturity matter more than language preference — Lua remains smaller, faster to start, and far more widely embedded.
- Your target is a one-chip MCU with tens of KB of RAM; even mruby is too large there (see mruby/c below).

## Alternatives

- lua/lua — use instead when you want the smallest, most battle-tested embeddable VM and language preference is negotiable; LuaJIT adds a JIT mruby lacks.
- ruby/ruby — use instead when you need full Ruby: gems, threads, stdlib, ecosystem; embedding CRuby is possible but heavyweight.
- mrubyc/mrubyc — mruby/c, an even smaller sibling VM for one-chip microcontrollers; use when RAM is measured in tens of KB.
- micropython/micropython — use instead when the team writes Python; comparable embedded-scripting niche with a larger hardware community.
- wren-lang/wren — use instead when you want a tiny class-based scripting language designed from scratch for embedding rather than adapted from a big language.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-04 | Repository published; METI-sponsored development[^1]. |
| 1.0.0 | 2014 | First stable release. |
| 2.0.0 | 2018-12 | Byte-oriented VM instruction encoding replaces 32-bit word opcodes; C API breaks[^4]. |
| 3.0.0 | 2021-03 | Preallocated symbols cut binary size; Ruby 3-era syntax adoption[^7]. |
| 3.x | 2022–2025 | Roughly annual releases (3.1–3.4) tracking CRuby syntax. |
| 4.0.0 | 2026 | Current stable line per README; Ruby 4.x syntax compatibility, amalgamation build[^1]. |

## References

[^1]: mruby README. https://github.com/mruby/mruby/blob/master/README.md
[^2]: ISO/IEC 30170:2012 — Ruby programming language standard. https://www.iso.org/standard/59579.html
[^3]: mruby documentation, "About the Limitations of mruby". https://github.com/mruby/mruby/blob/master/doc/limitations.md
[^4]: mruby internal docs, "About mruby Virtual Machine Instructions". https://github.com/mruby/mruby/blob/master/doc/internal/opcode.md
[^5]: mruby guide, "About GC Arena". https://github.com/mruby/mruby/blob/master/doc/guides/gc-arena-howto.md
[^6]: mruby guide, "ROM Method Tables for Memory-Efficient Method Registration". https://github.com/mruby/mruby/blob/master/doc/guides/rom-method-table.md
[^7]: mruby.org release archive. https://mruby.org/releases/

## Tags

c, ruby, embedded, scripting-language, interpreter, virtual-machine, microcontrollers, language-implementation, bytecode, sandboxing
