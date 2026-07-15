# pyinstaller/pyinstaller

> Freezes a Python program and its entire dependency tree — including the interpreter — into a self-contained executable.

[GitHub repo](https://github.com/pyinstaller/pyinstaller) ·
[Official website](https://pyinstaller.org) ·
[License: GPL-2.0-or-later with a bootloader exception](https://github.com/pyinstaller/pyinstaller/blob/develop/COPYING.txt)

## Overview

PyInstaller packages a Python application so it can run on a machine that has no
Python installed. It traces the imports of your entry script, collects every
module, shared library, and data file it can find, bundles a copy of the running
interpreter, and emits either a folder (`onedir`) or a single executable
(`onefile`). The lineage traces back to Gordon McMillan's Installer; the modern
project has been developed on GitHub since 2011[^1] and remains the default
answer to "how do I ship a Python desktop app to non-developers."

The defining property — and the source of most of its pain — is that PyInstaller
does static-plus-dynamic *analysis*, not compilation. It never runs your program
during the build; it walks the import graph with a modulegraph-style analyzer and
patches the gaps with a large library of per-package "hooks." This works
remarkably well for the common ecosystem (numpy, PyQt/PySide, matplotlib, and
hundreds of others ship working hooks out of the box) and fails silently at the
edges: anything imported by string, loaded via a plugin system, or discovered at
runtime is invisible to static analysis and must be declared by hand.

PyInstaller is explicitly **not a cross-compiler**. To produce a Windows binary
you must build on Windows; macOS on macOS; Linux on Linux (and effectively per
glibc/musl variant and architecture)[^2]. It also does not obfuscate or
meaningfully speed up code — the bundled `.pyc` is trivially recoverable. It is a
packager, not a protector and not a compiler.

## Getting Started

```bash
pip install pyinstaller
```

```bash
# onedir (default): a folder you distribute whole — faster start, easy to inspect
pyinstaller yourscript.py

# onefile: a single self-extracting executable — convenient, slower to launch
pyinstaller --onefile --name myapp yourscript.py

# GUI app with an icon, no console window (Windows/macOS)
pyinstaller --onefile --windowed --icon app.ico app.py
```

```python
# Resolving bundled data files at runtime — the canonical _MEIPASS pattern
import sys, os

def resource_path(rel):
    base = getattr(sys, "_MEIPASS", os.path.abspath("."))
    return os.path.join(base, rel)

logo = resource_path("assets/logo.png")   # works both frozen and unfrozen
```

The first `pyinstaller run` writes a `.spec` file — a Python script describing the
build. For anything beyond a trivial script you edit and re-run the spec directly
(`pyinstaller myapp.spec`) rather than passing flags each time.

## Architecture / How It Works

A build has two halves: **analysis** (Python) and the **bootloader** (C).

**Analysis.** `Analysis` parses the entry script's AST, follows imports
recursively, and builds a graph of pure-Python modules, C extensions, and their
transitive shared libraries. Hooks — one module per supported package under
`PyInstaller/hooks/` — inject the knowledge static analysis cannot derive: hidden
imports, data files, dynamic libraries, and metadata for packages like numpy or
PyQt. The results are serialized into a `PYZ` (a zlib-compressed ZIP of `.pyc`
bytecode) plus a `CArchive` (a custom archive of the interpreter, extension
modules, `.so`/`.dll`/`.dylib` dependencies, and data files).

**Bootloader.** The executable's entry point is not Python — it is a small C
program compiled per platform and architecture. On launch it initializes the
embedded CPython, mounts the `PYZ`, sets `sys._MEIPASS`, and runs your script.
Because it is compiled C, contributed/untested platforms require building the
bootloader from source (a C compiler and zlib headers), which `pip install`
attempts automatically when no prebuilt wheel matches[^2].

**onedir vs onefile.** In `onedir` the archive is unpacked on disk beside the
executable; startup is fast and the layout inspectable. As of 6.0 the support
files live in an `_internal/` subdirectory rather than cluttering the top
level[^3]. In `onefile` the CArchive is appended to the executable itself; at
each launch the bootloader extracts everything to a fresh temp directory
(`sys._MEIPASS`, e.g. `/tmp/_MEIxxxxxx`), runs from there, and deletes it on exit.
That extraction is why onefile binaries start slowly and why writing to files
"next to the exe" is a common bug — `_MEIPASS` is a throwaway temp path, not the
install location.

## Production Notes

**Antivirus false positives are the number-one operational headache.** onefile
executables share a self-extracting bootloader pattern with commodity malware
packers, so Windows Defender, SmartScreen, and various engines routinely flag or
quarantine freshly built binaries with no real threat present[^4]. There is no
clean fix from PyInstaller's side: mitigations are code-signing (Authenticode on
Windows, notarization on macOS), submitting false-positive reports to vendors,
preferring `onedir`, and occasionally rebuilding the bootloader locally so it does
not match a cached signature. Budget for this before shipping to end users.

**Startup latency.** onefile pays a decompress-and-extract tax on every launch —
noticeable (hundreds of ms to seconds) for large apps with heavy native
dependencies. If launch time matters, ship `onedir`.

**Hidden and dynamic imports.** `importlib.import_module("plugins." + name)`,
entry-point plugin systems, and conditional/optional imports are invisible to the
analyzer. Symptoms are `ModuleNotFoundError` only in the frozen build. Fixes:
`--hidden-import`, `--collect-all`/`--collect-submodules`, `datas`/`hiddenimports`
in the spec, or a custom hook. Third-party packages without a bundled hook may
need `pyinstaller-hooks-contrib`[^5] or a hand-written one.

**multiprocessing.** On Windows (spawn start method) and frozen apps generally,
`multiprocessing.freeze_support()` must be called at the top of `__main__` or the
app re-launches itself recursively.

**Binary size.** Bundling the interpreter plus native wheels routinely yields
tens to hundreds of MB. UPX compression (`--upx-dir`) reduces size but frequently
worsens antivirus flags and can corrupt some DLLs — treat it as opt-in and test.

**Reproducibility and CI.** Builds are host-specific; a matrix of runners
(one per OS/arch you ship) is unavoidable, and glibc version of the *build* host
sets the *minimum* glibc of the resulting Linux binary. Build on the oldest distro
you intend to support.

**Not security.** The bundled bytecode is extractable and decompilable. Do not
treat freezing as source protection.

## When to Use / When Not

**Use when:**
- You need to hand a double-clickable app to users who will not install Python.
- Your dependency set is mainstream (Qt/Tk GUI, numpy/pandas, requests) and has
  working hooks.
- You want a mature, well-trodden path with abundant Stack Overflow coverage.

**Avoid when:**
- You need a genuinely smaller or faster binary — Nuitka compiles to C and can
  both shrink and speed up code.
- You must cross-compile from one OS to another (PyInstaller cannot).
- You want to obfuscate or protect source (freezing does not).
- You are shipping a pure library or a server app that lives in a container — a
  wheel or a base-image `pip install` is simpler than freezing.

## Alternatives

- Nuitka/Nuitka — compiles Python to C and links a real binary; smaller/faster
  output, harder to reverse, slower and pickier builds. Use when performance or
  binary size matters.
- marcelotduarte/cx_Freeze — the long-standing setuptools-integrated freezer; use
  when you prefer a `setup.py`-driven build and simpler internals.
- beeware/briefcase — packages apps into native installers (msi/dmg/deb) and
  targets mobile; use when you want distributable installers and cross-platform
  GUI packaging, not a bare exe.
- indygreg/PyOxidizer — embeds Python in a Rust launcher, can run modules from
  memory; powerful but effectively unmaintained now, so use with caution.
- conda/constructor — builds installers from conda environments; use when your
  stack is already conda and you ship scientific/native-heavy apps.

## History

| Version | Date | Notes |
|---------|------|-------|
| (project on GitHub) | 2011-11-23 | Repo created; descends from McMillan Installer[^1]. |
| 3.0 | 2015-10 | Python 3 support merged into mainline. |
| 4.0 | 2020-08 | Python 2 support dropped; Python 3-only. |
| 5.0 | 2022-01 | Packaging modernized; hooks split to `pyinstaller-hooks-contrib`[^5]. |
| 6.0 | 2023-09 | onedir output restructured under `_internal/`; hook and bootloader overhaul[^3]. |
| 6.21.0 | 2026 | Current 6.x line; supports CPython 3.8–3.15[^6]. |

## References

[^1]: PyInstaller repository, created 2011-11-23. https://github.com/pyinstaller/pyinstaller
[^2]: PyInstaller docs, "Requirements" and "Supporting Multiple Operating Systems" — not a cross-compiler; bootloader is compiled per platform. https://pyinstaller.org/en/stable/requirements.html
[^3]: PyInstaller 6.0 changelog — onedir contents moved into an `_internal` directory. https://pyinstaller.org/en/stable/CHANGES.html
[^4]: PyInstaller docs, "When Things Go Wrong" — antivirus false-positive guidance. https://pyinstaller.org/en/stable/when-things-go-wrong.html
[^5]: pyinstaller-hooks-contrib — community hooks for third-party packages. https://github.com/pyinstaller/pyinstaller-hooks-contrib
[^6]: PyInstaller README — supported Python versions 3.8–3.15. https://github.com/pyinstaller/pyinstaller

## Tags

python, packaging, executable, freezer, cross-platform, desktop-apps, bundler, distribution, windows, macos, linux, cli
