# pyodide/pyodide

> A full CPython interpreter compiled to WebAssembly, plus a scientific Python stack and a two-way JavaScript FFI, running in the browser and Node.js.

[GitHub repo](https://github.com/pyodide/pyodide) ·
[Official website](https://pyodide.org/en/stable/) ·
[License: MPL-2.0](https://github.com/pyodide/pyodide/blob/main/LICENSE)

## Overview

Pyodide is a port of CPython to WebAssembly via Emscripten[^1]. Unlike Python-to-JS transpilers, it runs the actual reference interpreter, so language semantics, the standard library, and C-extension behavior match desktop CPython closely. It was created in 2018 by Michael Droettboom at Mozilla as part of the Iodide notebook project[^2]; Iodide was archived, and Pyodide continued as an independent, community-governed project under its own GitHub organization.

The reason to reach for Pyodide is the scientific stack: NumPy, pandas, SciPy, Matplotlib, and scikit-learn are pre-built against the Emscripten ABI and ship as loadable WebAssembly packages. Pure-Python wheels from PyPI can be installed at runtime with `micropip`. This is what separates it from lighter in-browser Python options — you get real NumPy, not a reimplementation.

The defining tradeoff is weight. You are shipping a C interpreter, its standard library, and megabytes of numerical libraries into a browser tab. The core runtime download is on the order of several megabytes even before packages, and cold startup involves fetching, compiling, and instantiating WebAssembly plus unpacking a virtual filesystem. Pyodide is the right tool when you specifically need CPython-compatible execution client-side; it is the wrong tool when a smaller interpreter or a server round-trip would do.

## Getting Started

Load from a CDN in the browser:

```html
<script src="https://cdn.jsdelivr.net/pyodide/v0.27.0/full/pyodide.js"></script>
<script type="module">
  const pyodide = await loadPyodide();
  await pyodide.loadPackage("numpy");
  const result = pyodide.runPython(`
    import numpy as np
    float(np.arange(10).mean())
  `);
  console.log(result); // 4.5
</script>
```

In Node.js:

```bash
npm install pyodide
```

```js
import { loadPyodide } from "pyodide";
const pyodide = await loadPyodide();
// install a pure-Python wheel from PyPI at runtime
await pyodide.loadPackage("micropip");
const micropip = pyodide.pyimport("micropip");
await micropip.install("snowballstemmer");
console.log(pyodide.runPython("import snowballstemmer; 'ok'"));
```

## Architecture / How It Works

Pyodide is not a single artifact but a set of components[^3]:

1. **A patched CPython build** compiled to WebAssembly with Emscripten. The patches adapt CPython to the single-threaded, no-fork, no-subprocess constraints of the WASM/Emscripten environment.
2. **A JavaScript ⟺ Python FFI** (`src/core`, `src/py`). Python objects surface in JS as `PyProxy` objects; JS objects surface in Python as `JsProxy`. Type translation covers numbers, strings, buffers, callables, async/await (Python awaitables become JS promises and vice versa), and exceptions.
3. **A JavaScript loader** (`src/js`) that fetches the WASM binary, sets up the Emscripten virtual filesystem (MEMFS), and exposes the `loadPyodide` / `loadPackage` / `runPython` API.
4. **An Emscripten "platform"** — a specific Emscripten version plus ABI-sensitive compile flags and a set of static libraries. Every binary package must be built against the exact same platform, which is why the ABI is versioned explicitly[^4].
5. **A separate build toolchain** — `pyodide-build` (cross-compilation), `pytest-pyodide` (testing), and `micropip` (runtime install) live in their own repositories.

Packages come in two flavors. Prebuilt packages in the distribution are C/C++/Rust extensions cross-compiled to the Emscripten ABI and loaded with `pyodide.loadPackage`. Pure-Python packages can be pulled from PyPI at runtime with `micropip`, because a pure wheel has no native code to rebuild. A package with C extensions that isn't in the distribution cannot simply be `pip install`ed — it has to be compiled against the matching Emscripten toolchain first.

Execution is single-threaded. WebAssembly threads exist but require cross-origin isolation headers (`COOP`/`COEP`) and are not the default execution model; most Pyodide code runs on one thread, and blocking Python code blocks the browser event loop unless you run Pyodide in a Web Worker.

## Production Notes

**Download and startup cost.** The core runtime plus stdlib is several megabytes; adding NumPy/pandas/SciPy pushes the total well past that. Budget for a visible load delay on first visit. Serve with compression (the assets pre-compress well) and cache aggressively — the versioned CDN paths are immutable, so long cache lifetimes are safe.

**Run it in a Web Worker.** Python calls are synchronous and CPU-bound. On the main thread they freeze the UI. The standard production pattern is to host Pyodide in a Web Worker and message-pass results back, keeping the main thread responsive.

**FFI memory management is manual at the boundary.** A `PyProxy` created in JS holds a reference to a Python object; the JS garbage collector does not know how to free it. Long-lived proxies must be released with `.destroy()`, or you leak Python-side memory. This is the most common source of subtle leaks in Pyodide apps.

**ABI pinning is strict.** Binary packages are only compatible with the Pyodide version (and Emscripten platform) they were built for. Mixing a package built for one release into another's runtime fails or crashes. Pin the Pyodide version and load packages from the matching distribution; do not assume `pip`-style version independence.

**No subprocesses, no sockets, no threads-by-default.** The Emscripten sandbox has no `fork`/`exec`, no raw sockets, and a restricted filesystem. Code that shells out, spawns processes, or opens TCP connections will not work unmodified. Networking goes through the browser's `fetch`, exposed via the FFI, not through `socket`.

**Version cadence.** Pyodide has stayed on a `0.x` line and ships breaking changes between minor releases (FFI behavior, packaging, bundled CPython version). Upgrades are not drop-in — read the changelog, and expect to re-test the FFI surface and re-pin package loads on each bump.

## When to Use / When Not

**Use when:**
- You need real CPython semantics and the scientific stack (NumPy/pandas/SciPy) running client-side, offline, or without a Python backend.
- You are building interactive notebooks, data-viz sandboxes, in-browser REPLs, or teaching tools.
- You want to reuse existing Python code in the browser rather than rewrite it in JS.

**Avoid when:**
- Payload size or fast cold start matters more than compatibility (a landing page, a mobile-first app on flaky networks).
- Your workload is trivial scripting that a smaller interpreter or plain JS could handle.
- You need threads, subprocesses, or raw sockets, or a C-extension package that isn't in the distribution.
- A server-side Python process is available and latency is acceptable — running CPython on a server is simpler and lighter for the client.

## Alternatives

- pyscript/pyscript — an HTML-first app framework built on top of Pyodide (and MicroPython); use it when you want to embed Python in a page declaratively rather than drive the low-level FFI yourself.
- brython/brython — transpiles Python to JavaScript; use when you want small payloads and DOM scripting and can live without C extensions or CPython-exact semantics.
- micropython (WASM build) — a much smaller, faster-starting interpreter; use when startup weight dominates and you don't need the scientific stack or full stdlib.
- RustPython/RustPython — a Python interpreter written in Rust that targets WASM; use for a lighter interpreter when incomplete stdlib coverage is acceptable.
- skulpt/skulpt — a Python-in-JavaScript implementation aimed at education; use for simple teaching sandboxes, not compatibility.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018 | Created at Mozilla inside the Iodide project[^2]. |
| 0.17 | 2021-04 | Major refactor: faster load, restructured FFI, spun out as an independent project[^5]. |
| 0.21 | 2022-09 | `pyodide-build` split out; distribution/packaging overhaul. |
| 0.23 | 2023-04 | Bundled CPython advanced; Node.js support hardened. |
| 0.25 | 2024-01 | Explicit Emscripten ABI versioning documented[^4]. |
| 0.26 | 2024-06 | CPython 3.12 base; further package and FFI updates. |
| 0.27 | 2024-12 | Continued CPython/Emscripten updates and packaging changes. |

## References

[^1]: Pyodide README — "Pyodide is a port of CPython to WebAssembly/Emscripten." https://github.com/pyodide/pyodide
[^2]: Pyodide README, History section — created 2018 by Michael Droettboom at Mozilla as part of Iodide. https://github.com/pyodide/pyodide
[^3]: Pyodide README, "The Components of the Pyodide Project." https://github.com/pyodide/pyodide
[^4]: Pyodide docs — Emscripten platform / ABI. https://pyodide.org/en/stable/development/abi.html
[^5]: Pyodide blog and changelog. https://blog.pyodide.org/ and https://pyodide.org/en/stable/project/changelog.html

## Tags

python, webassembly, wasm, emscripten, cpython, browser, scientific-computing, numpy, javascript-ffi, micropip, nodejs
