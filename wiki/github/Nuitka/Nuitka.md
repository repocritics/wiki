# Nuitka/Nuitka

> A Python-to-C compiler that packages a Python program into a standalone executable while staying bug-for-bug compatible with CPython.

[GitHub repo](https://github.com/Nuitka/Nuitka) ·
[Official website](https://nuitka.net) ·
[License: AGPL-3.0](https://github.com/Nuitka/Nuitka/blob/develop/LICENSE.txt)

## Overview

Nuitka is a whole-program compiler for Python, itself written in Python, maintained principally by Kay Hayen since 2012[^1]. It reads your Python source, translates it into C that calls into `libpython` and Nuitka's own static C runtime, and hands the C off to a real C compiler (gcc, clang, MSVC, MinGW64, or zig). The output is a native executable or a CPython extension module that runs the same program. It supports an unusually wide version range — CPython 2.6, 2.7, and 3.4 through 3.14 — compiling with whatever interpreter you run it under[^2].

The defining thing to understand about Nuitka is what it is *not*: it is not a JIT and not a type-specializing optimizer. Unlike Cython (which rewards type annotations) or PyPy (which traces hot loops), Nuitka's goal is faithful CPython semantics, not aggressive speedup. Real-world programs typically see modest gains, sometimes none, because most of the time is still spent in `libpython` and C extensions that Nuitka does not rewrite. The payoff is elsewhere: a single distributable artifact, mild acceleration, and source that ships as compiled C rather than readable `.pyc`.

That places Nuitka in two overlapping markets at once — the *packaging* market (PyInstaller, cx_Freeze, py2app) and the *acceleration* market (Cython, mypyc, Codon). It is one of the few tools that credibly does both, which is also why its documentation and flag surface are large. Development is active, with releases roughly monthly and a commercial edition that funds the project.

## Getting Started

```bash
python -m pip install Nuitka
```

Compile and run a script (acceleration mode still depends on your installed CPython):

```bash
# hello.py
python -m nuitka --follow-imports hello.py
./hello.bin        # hello.exe on Windows
```

Build a self-contained distribution that runs on a machine without Python:

```bash
# A folder you can copy anywhere:
python -m nuitka --mode=standalone hello.py     # -> hello.dist/

# A single extracting executable:
python -m nuitka --mode=onefile hello.py        # -> hello.bin / hello.exe
```

Nuitka will offer to download a MinGW64 toolchain and a C caching tool on first run if no suitable compiler is found — accept both.

## Architecture / How It Works

Nuitka is a multi-stage source-to-source-to-native pipeline:

1. **Parse and build a tree** — Python source is parsed and lowered into Nuitka's own node tree, distinct from CPython's AST.
2. **Optimize** — a fixpoint optimizer runs constant folding, function inlining, and limited value/type propagation over that tree. This is where the "clever things" happen, but the scope is deliberately conservative to preserve semantics.
3. **Generate C** — the tree is emitted as C source that manipulates `PyObject*` values, calling `libpython` for most operations and Nuitka's static helper C for the rest.
4. **Compile and link** — an internally bundled copy of **Scons** orchestrates the actual C compilation. This is why Nuitka has a compile-time dependency on a second Python for CPython 3.4 (Scons does not run there) and why the C compiler choice matters.

The output *mode* determines coupling:

- **Acceleration** (default) — a native binary that still imports from your installed CPython and its site-packages. Not portable.
- **Module / package** — a `.so`/`.pyd` extension you import in place of the `.py`. Version-locked to the CPython it was built against.
- **Standalone** — a `.dist/` folder bundling the interpreter, the compiled program, and every followed dependency. Portable to same-OS machines.
- **Onefile** — standalone compressed into one executable that **extracts to a temp directory at startup** and runs from there, paying an unpack cost on each launch.

Framework-specific knowledge (which data files, DLLs, and hidden imports a package needs) lives in Nuitka's **plugin** system — `--enable-plugin=pyside6`, `tk-inter`, `anti-bloat`, etc. Getting a large app to package correctly is largely a matter of finding the right plugins and `--include-data-*` flags.

## Production Notes

**Licensing is the first thing to check.** Nuitka is AGPL-3.0, unusual for a build tool, but ships a **runtime exception** (`LICENSE-RUNTIME.txt`) so the *binaries you produce* are not encumbered by the AGPL[^3]. You can ship closed-source products. The AGPL applies to Nuitka's own source, not your compiled output. Read the exception text before assuming.

**Compile times and toolchain friction.** Real C compilation of a whole standalone app is slow — minutes to tens of minutes — and requires a working C compiler. On Windows, Nuitka insists on *its own* MinGW64 download (to avoid the toolchain breakage it has repeatedly hit), and MinGW64 does **not** support Python 3.13+, pushing you to MSVC there. The C caching tool (ccache/clcache) matters a lot for iteration speed.

**Interpreter provenance is load-bearing.** Nuitka is tied to CPython implementation details: Windows Store Python does not work (it is checked against), and macOS `pyenv`-built Python is known bad — use Homebrew or python.org, or standalone builds will be less backward-compatible. Anaconda and Homebrew CPython are supported.

**Standalone/onefile packaging is where projects actually get stuck.** Data files are not code and are not followed automatically — you must declare them with `--include-data-files`/`--include-data-dir`, and shipping `.py`/`.pyc` as "data" silently fails (Nuitka trims stdlib it cannot see imported). Onefile's temp-dir extraction interacts badly with antivirus, `__file__`-relative path assumptions, and multi-process apps; test `--mode=standalone` first because its failures are easier to diagnose.

**Debugging.** Because the program is now C, tracebacks and tooling differ from interpreted runs. Reproduce and fix logic bugs under plain `python` before compiling — Nuitka does not make broken code easier to debug.

**Commercial edition.** Data-file encryption, DLL/import protection, and stronger source obfuscation are paid features[^4]. Plain Nuitka gives you compiled C (harder to read than `.pyc`) but not true obfuscation.

## When to Use / When Not

**Use when:**
- You need to ship a Python app to users who have no Python, and want one artifact.
- You want compiled-C source concealment beyond stripped `.pyc`.
- You value CPython-exact behavior and broad version support over maximum speed.
- You want a packaging path that also yields some runtime acceleration for free.

**Avoid when:**
- Your goal is large speedups on numeric/hot-loop code — Cython, mypyc, Codon, or PyPy target that directly.
- You need fast, iterative packaging in CI with no C toolchain — PyInstaller has less build friction.
- You are on Python 3.13+ on Windows and cannot install MSVC.
- Your app leans on dynamic/`__import__` magic that a static compiler cannot trace without manual `--include-*` flags.

## Alternatives

- pyinstaller/pyinstaller — use instead when you want simpler, faster packaging with no C compiler and don't need compiled source.
- marcelotduarte/cx_Freeze — use instead for a lighter, setuptools-integrated freezer without native compilation.
- cython/cython — use instead when you want real speedups by annotating types in hot modules, not whole-program packaging.
- mypyc/mypyc — use instead when your code is already type-annotated and you want ahead-of-time compilation of it.
- exaloop/codon — use instead when you accept a Python-*like* language for aggressive, LLVM-based performance rather than full CPython compatibility.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012 | First public release by Kay Hayen; GitHub repo created 2013[^1]. |
| 0.6.0 | 2018 | Long 0.x line; C11 backend, plugin system matured. |
| 1.0 | 2022 | First "1.0" milestone after a decade of 0.x releases[^5]. |
| 2.0 | 2024 | Major line; `--mode=` flags consolidate onefile/standalone/module. |

Version-specific dates beyond the year are omitted where not verified; releases have been roughly monthly for years.

## References

[^1]: Nuitka home and about — Kay Hayen, project since 2012. https://nuitka.net
[^2]: Nuitka User Manual — supported Python (2.6, 2.7, 3.4–3.14) and C compiler requirements. https://nuitka.net/user-documentation/
[^3]: Nuitka licensing — AGPL-3.0 with runtime exception (`LICENSE.txt` / `LICENSE-RUNTIME.txt`). https://github.com/Nuitka/Nuitka/blob/develop/LICENSE.txt
[^4]: Nuitka Commercial — paid data-file protection and obfuscation features. https://nuitka.net/doc/commercial/
[^5]: Nuitka release posts. https://nuitka.net/posts/

## Tags

python, compiler, python-compiler, packaging, standalone-executable, native-compilation, c-backend, distribution, agpl, cross-platform
