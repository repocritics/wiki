# wren-lang/wren

> A small, fast, class-based concurrent scripting language for embedding — "Smalltalk in a Lua-sized package."

[GitHub repo](https://github.com/wren-lang/wren) ·
[Official website](http://wren.io) ·
[License: MIT](https://github.com/wren-lang/wren/blob/main/LICENSE)

## Overview

Wren is an embeddable scripting language written by Bob Nystrom (author of *Game Programming Patterns* and *Crafting Interpreters*), started in late 2013[^1]. It occupies the same niche as Lua — a dependency-free C VM you compile into a host application — but makes different language choices: classes are the core abstraction (Lua has tables and metatables), and lightweight cooperative fibers are built into the execution model rather than bolted on. The entire VM is under 4,000 semicolons of deliberately readable, heavily commented C99[^2], which makes it one of the most-cited codebases for learning how real bytecode VMs work.

Day-to-day stewardship passed from Nystrom to community maintainers around the 0.2.0 release in 2019[^3]. That transition tells the honest story of the project: the language design is essentially complete and stable, but the pace is glacial. The last tagged release is 0.4.0 (April 2021)[^4]; the main branch still receives occasional fixes (last push November 2025), with ~270 open issues. At ~8,100 stars it has mindshare, but it is a finished artifact more than a moving project — which for an embeddable language is arguably a feature, since embedders want a frozen target, and a liability, since bugs and platform quirks are fixed slowly or not at all.

## Getting Started

Wren ships as a library, not an application. Build the static library and embed it (the standalone interpreter lives in the separate `wren-lang/wren-cli` repo):

```bash
git clone https://github.com/wren-lang/wren
cd wren/projects/make    # premake-generated makefiles per platform
make                     # produces lib/libwren.a
```

Minimal C host:

```c
#include "wren.h"

int main() {
  WrenConfiguration config;
  wrenInitConfiguration(&config);
  WrenVM* vm = wrenNewVM(&config);
  wrenInterpret(vm, "main", "System.print(\"Hello from Wren\")");
  wrenFreeVM(vm);
  return 0;
}
```

Wren itself is class-based with fibers:

```dart
class Bird {
  flyTo(city) { System.print("Flying to %(city)") }
}

var f = Fiber.new {
  ["a", "b", "c"].each {|x| Fiber.yield(x) }
}
while (!f.isDone) System.print(f.call())
```

## Architecture / How It Works

Wren follows the classic Lua playbook with its own twists[^2]:

- **Single-pass compiler.** Source compiles directly to bytecode with no AST — one reason startup and compile are fast, and one reason error messages and tooling hooks are limited. There is nowhere to hang a type checker or formatter.
- **Stack-based bytecode VM** with computed-goto dispatch on compilers that support it, falling back to a switch loop elsewhere.
- **NaN tagging.** All values fit in 64-bit doubles; pointers, booleans, and null are smuggled into quiet-NaN bit patterns (`WREN_NAN_TAGGING`, optional for platforms where it misbehaves). The commentary in `wren_value.h` explaining this is a minor classic[^5].
- **Signature-based dispatch.** Methods are identified by name *and* arity (`add(_,_)` is distinct from `add(_)`), interned into a global symbol table. Dispatch is an indexed array lookup into a class's method table rather than a string-keyed hash lookup — a large part of why Wren benchmarks competitively with Lua's interpreter[^6].
- **Fibers everywhere.** Every Wren program runs inside a fiber; fibers have small, dynamically grown stacks, and switching is a VM-internal operation, not an OS context switch. Error handling is fiber-based too: a runtime error aborts the current fiber, and `Fiber.try` is the closest thing to try/catch. There are no exceptions in the Java/Python sense.
- **Slot-based embedding API.** Host and VM exchange values through numbered slots (`wrenGetSlotDouble`, `wrenSetSlotHandle`), and the host registers foreign methods and foreign classes (with finalizers) via binding callbacks. Module loading is delegated to host callbacks, so the embedder fully controls what `import` means.
- **Mark-sweep GC**, stop-the-world, tunable via `initialHeapSize` / `heapGrowthPercent` in the VM configuration. No incremental or generational mode.

The VM has no global state — each `WrenVM` is isolated, so multiple VMs can coexist, one per thread. A single VM is not thread-safe.

## Production Notes

- **Pin a commit.** With no release since 0.4.0 (2021) and fixes landing only on main, serious embedders vendor the source at a known commit rather than tracking tags. The upside: upgrade churn is essentially zero.
- **The core library is intentionally tiny.** No file I/O, no networking, no OS access in the VM — those live in wren-cli or in your host bindings. This is a sandboxing advantage (scripts can only do what you expose) but means every project rebuilds the same glue. There is no package manager or module ecosystem to speak of.
- **All numbers are doubles.** No integer type; exact integers hold to 2^53 and bitwise operators truncate to 32 bits. Fine for game scripting, a trap for IDs and money.
- **GC pauses.** The collector is stop-the-world mark-sweep. For typical embedded workloads (small heaps, per-frame scripts) this is a non-issue; large object graphs will produce visible pauses with no incremental option.
- **Tooling is thin.** No debugger protocol, no official LSP, limited error recovery from the single-pass compiler. Community analyzers exist but are stale. Budget for `System.print` debugging.
- **Maintenance risk is real.** Issues can sit for years; if you hit a VM bug you should be prepared to patch it yourself. The codebase being small and readable is the mitigating factor — self-support is genuinely feasible here in a way it is not with most language runtimes.
- Notable adopters are mostly indie game tooling: the DOME game framework builds its scripting layer on Wren[^7], and it appears in various custom engines. There is no large corporate deployment to anchor long-term investment.

## When to Use / When Not

**Use when:**
- You are embedding scripting in a C/C++ application (games especially) and want a class-based object model instead of Lua's tables.
- You want built-in coroutines/fibers as the concurrency primitive for script-driven game logic.
- You value a runtime small enough to audit, patch, and sandbox yourself.
- You want a readable real-world VM to study alongside *Crafting Interpreters*.

**Avoid when:**
- You need an ecosystem — packages, debuggers, IDE support, Stack Overflow volume. Lua dwarfs Wren on all counts.
- You need JIT-level performance; Wren is a fast interpreter, not a JIT.
- You need integers, a rich stdlib, or OS APIs out of the box.
- You need an actively responsive upstream with regular releases.

## Alternatives

- lua/lua — the default embeddable scripting choice; use it when ecosystem, documentation, and hiring familiarity outweigh language aesthetics.
- LuaJIT/LuaJIT — use when script execution speed is the constraint; its trace JIT is in a different performance class than any interpreter.
- mruby/mruby — embeddable Ruby with mrbgems; use when you want a richer language and stdlib and can accept a larger runtime.
- marcobambini/gravity — closest design sibling (small, class-based, embeddable, C99); similar low-activity risk profile.
- albertodemichelis/squirrel — C++-friendly with game-industry pedigree (Valve shipped it); use for tighter C++ integration idioms.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013-11 | Repository created; initial development by Bob Nystrom[^1]. |
| 0.1.0 | 2016-05 | First tagged version. |
| 0.2.0 | 2019-10 | First formal release under community maintenance[^3]. |
| 0.3.0 | 2020-06 | CLI split out to wren-lang/wren-cli; core repo becomes VM-only[^8]. |
| 0.4.0 | 2021-04 | Last release to date: attributes, `as` import renaming, API additions[^4]. |

## References

[^1]: GitHub repository metadata — created 2013-11-19. https://github.com/wren-lang/wren
[^2]: Wren README — "The VM implementation is under 4,000 semicolons." https://github.com/wren-lang/wren#readme
[^3]: Wren 0.2.0 release notes — 2019-10-01. https://github.com/wren-lang/wren/releases/tag/0.2.0
[^4]: Wren 0.4.0 release notes — 2021-04-09. https://github.com/wren-lang/wren/releases/tag/0.4.0
[^5]: NaN-tagging commentary in `wren_value.h`. https://github.com/wren-lang/wren/blob/main/src/vm/wren_value.h
[^6]: Wren performance page — method dispatch and benchmark methodology. http://wren.io/performance.html
[^7]: DOME — game framework scripted with Wren. https://domeengine.com/
[^8]: Wren 0.3.0 release notes — 2020-06-05. https://github.com/wren-lang/wren/releases/tag/0.3.0

## Tags

wren, c, scripting-language, embeddable, bytecode-vm, interpreter, fibers, game-development, class-based, coroutines
