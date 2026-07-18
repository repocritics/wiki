# LuaJIT/LuaJIT

> A trace-based Just-In-Time compiler for Lua 5.1 — one of the fastest dynamic
> language implementations ever built, maintained by essentially one person.

[GitHub repo](https://github.com/LuaJIT/LuaJIT) ·
[Official website](https://luajit.org) ·
[License: MIT](https://github.com/LuaJIT/LuaJIT/blob/v2.1/COPYRIGHT)

## Overview

LuaJIT is Mike Pall's JIT-compiling implementation of Lua 5.1, started in 2005
and rewritten as a trace compiler for the 2.0 series[^1]. It routinely runs Lua
code 10–50× faster than the reference PUC-Rio interpreter and, on numeric
workloads that stay on-trace, approaches C. It powers the OpenResty/Kong nginx
ecosystem, Neovim's scripting layer, the LÖVE game framework, and Tarantool
(via a fork), which makes it one of the most production-deployed JITs outside
of browser engines.

The defining tension: LuaJIT is frozen at Lua 5.1 semantics (plus selected
5.2/5.3 extensions) and is developed by a single author. Mike Pall has stated
the project is in long-term maintenance; full Lua 5.2+ compatibility is
explicitly not a goal[^2]. Users trade language evolution and bus-factor risk
for exceptional performance and a stable target. The GitHub repo is a mirror
of the author's tree — 5,650 stars understate its footprint, since most of its
users consume it indirectly through OpenResty, Neovim, or game engines.

Since 2023 there are no versioned releases: the `v2.1` branch head is the
release[^3]. Activity is steady (last push 2026-07) but consists of fixes and
ports, not new features.

## Getting Started

No official binary releases — build from source (seconds, no dependencies):

```bash
git clone https://luajit.org/git/luajit.git
cd luajit
make && sudo make install
```

LuaJIT is a drop-in `lua` replacement plus the FFI, its second killer feature —
declare C signatures inline and call shared libraries with near-zero overhead:

```lua
-- hello.lua
local ffi = require("ffi")
ffi.cdef[[
  int printf(const char *fmt, ...);
]]
ffi.C.printf("Hello from %s\n", "LuaJIT")

-- FFI structs are raw C data, not tables: compact and JIT-friendly
local point = ffi.new("struct { double x, y; }", 3, 4)
print(math.sqrt(point.x^2 + point.y^2))  --> 5
```

Run with `luajit hello.lua`. Use `luajit -jv` (verbose trace log) or
`-jdump` to see what the compiler is doing.

## Architecture / How It Works

LuaJIT is a two-tier VM[^4]:

1. **Interpreter** — hand-written in assembler via DynASM (Pall's own
   preprocessor), with its own compact bytecode format. Even with the JIT
   disabled (`-joff`), this interpreter is several times faster than PUC Lua.
2. **Trace compiler** — hot loops and calls (detected by counters) are
   *recorded*: the interpreter logs the actual instructions executed along one
   concrete control-flow path into an SSA IR. The IR is optimized (constant
   folding, loop-invariant code motion, allocation sinking, dead-store
   elimination) and emitted as machine code. Guards embedded in the trace
   verify the recorded assumptions (types, targets); a failed guard exits to
   the interpreter, and hot side exits spawn attached side traces.

Consequences of trace compilation:

- **NYI aborts.** Operations the recorder does not support (some standard
  library functions, `pairs` historically, certain patterns) abort the trace;
  code containing them in a hot loop stays interpreted[^5]. Performance is
  therefore bimodal: on-trace code is extremely fast, blacklisted code is not.
- **FFI data is JIT-native.** `ffi.cdef` types compile to direct machine loads
  and stores; idiomatic high-performance LuaJIT uses C structs, not tables.
- **GC is Lua 5.1's incremental collector.** The redesigned GC sketched for
  3.0 was never built; allocation-heavy loads bottleneck on it.

Ports exist for x86/x64, ARM, ARM64, PPC, MIPS, and (in the rolling 2.1
branch) RISC-V. The `LJ_GC64` build mode uses full 64-bit GC references and
is required on ARM64.

## Production Notes

- **Memory ceiling.** Classic x64 builds without `LJ_GC64` confine all GC
  objects to the low 2 GB of address space; large-heap services hit "not
  enough memory" long before the machine is exhausted. Verify build flags
  before deploying memory-hungry workloads.
- **Performance is fragile at the edges.** A single NYI call or overlong trace
  in a hot path can silently drop you to interpreter speed. Operators profile
  with `-jv`, `-jdump`, and `jit.p`; OpenResty culture treats trace-abort
  hunting as routine tuning work.
- **Sole-maintainer risk is real, priced in.** Fixes land, but review
  bandwidth is one person. Heavy users hedge with forks (OpenResty's luajit2,
  Tarantool's); upstream vs. luajit2 is a real decision for nginx stacks.
- **Language freeze.** Code written for Lua 5.3/5.4 (integer semantics,
  native bitwise operators, `utf8`) does not run unmodified; the supported
  dialect is 5.1 plus documented extensions[^2]. Check before adopting
  libraries.
- **No versioned upgrades.** Rolling releases mean "pin a commit hash" is the
  only reproducible deployment strategy; distro-packaged versions are
  arbitrary snapshots and diverge significantly.
- **Coroutine/C boundary.** Yielding across a C call is restricted (a Lua 5.1
  inheritance); mixing C callbacks and coroutines in embeddings needs care.

## When to Use / When Not

**Use when:**
- You need a fast embeddable scripting language in a C/C++ host (games,
  proxies, databases) and Lua 5.1 semantics are acceptable.
- You are on the OpenResty/nginx stack — LuaJIT is the assumed runtime.
- You need cheap C interop: the FFI beats the classic Lua/C API in both
  ergonomics and speed.
- Latency matters: startup is instant and the runtime is a few hundred KB.

**Avoid when:**
- You need Lua 5.4 language features or want to track upstream Lua evolution.
- Your workload is allocation-heavy and GC-bound — the JIT cannot help there.
- You need sandboxed untrusted code with strong guarantees; the FFI is a
  process-takeover primitive and must be firewalled off.
- Single-maintainer dependency risk is unacceptable to your organization.

## Alternatives

- lua/lua — the reference PUC-Rio implementation; use it when you want
  current Lua (5.4), maximum portability, and simplicity over speed.
- luau-lang/luau — Roblox's Lua 5.1 descendant with gradual typing and a fast
  non-JIT interpreter; better for sandboxed untrusted scripts.
- openresty/luajit2 — OpenResty's maintained fork with extra APIs and
  tuning; the default choice inside nginx/OpenResty deployments.
- tarantool/luajit — Tarantool's fork with its own GC and platform work;
  relevant only inside the Tarantool ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2005 | First LuaJIT generation (method-at-a-time design)[^1]. |
| 2.0.0-beta1 | 2009 | Complete rewrite: trace compiler, DynASM interpreter. |
| 2.0.0 | 2012-11 | First stable 2.0 release. |
| 2.1.0-beta1 | 2015-08 | ARM64 port, `LJ_GC64` mode. |
| 2.1.0-beta3 | 2017-05 | Last tagged release. |
| rolling | 2023– | Versioned releases abandoned; `v2.1` branch head is the release[^3]. |

## References

[^1]: LuaJIT project history and changelog. https://luajit.org/changelog.html
[^2]: LuaJIT extensions — supported 5.2/5.3 features. https://luajit.org/extensions.html
[^3]: LuaJIT project status — rolling release model. https://luajit.org/status.html
[^4]: Mike Pall, LuaJIT 2.0 design notes (trace compiler overview). http://lua-users.org/lists/lua-l/2009-11/msg00089.html
[^5]: LuaJIT wiki, "NYI" — operations not compiled by the trace recorder. https://web.archive.org/web/2023/http://wiki.luajit.org/NYI

## Tags

c, lua, jit-compiler, virtual-machine, ffi, embedded-scripting, tracing-jit, performance, openresty, language-runtime
