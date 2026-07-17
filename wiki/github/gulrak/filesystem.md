# gulrak/filesystem

> A header-only, C++11/14/17/20 backport of C++17's `std::filesystem` — the drop-in you reach for when your toolchain's own `<filesystem>` isn't available.

[GitHub repo](https://github.com/gulrak/filesystem) ·
[License: MIT](https://github.com/gulrak/filesystem/blob/master/LICENSE)

## Overview

`gulrak/filesystem` (namespace `ghc::filesystem`) is an implementation of the C++17 `std::filesystem` API that compiles under C++11, C++14, C++17, and C++20[^1]. It exists to solve a specific transitional problem: `std::filesystem` was standardized in C++17, but real toolchains shipped it late and inconsistently — libstdc++ needed a separate `-lstdc++fs`, libc++ needed `-lc++fs` and gated it behind macOS deployment targets (unavailable before macOS 10.15 / iOS 13), and codebases stuck on C++11/14 had no standard option at all. This library gives those codebases a single `.hpp` to include with a nearly identical interface.

The author (Steffen Schümann; "ghc" = "gulrak's helper classes", explicitly *not* Haskell's GHC[^2]) frames it modestly: it does not try to be a "better" `std::filesystem`, only an almost drop-in one for environments that can't use the real thing. The one deliberate divergence from the standard is encoding: the library follows the "UTF-8 Everywhere" philosophy, so `std::string` values are treated as UTF-8 (i.e. as if `std::u8string`) and `std::u16string` as UTF-16, on all platforms including Windows[^3]. That is the defining tension — it fixes Windows' Unicode path pain by breaking strict conformance with `std::filesystem`'s implementation-defined narrow-string encoding, so a mixed `ghc`/`std` codebase can behave differently on the same path.

Because the standard problem it addresses is fading — every mainstream compiler now ships a working `<filesystem>` — the library's role has shifted from "the way to get filesystem" to "the way to keep supporting old platforms." It remains actively maintained: 1,550 stars, 203 forks, last push 2026-07-14, latest release v1.5.14.

## Getting Started

There is no build step for basic use — copy `include/ghc/filesystem.hpp` into your project, or add the repo as a submodule and point your include path at it.

```cpp
#include <ghc/filesystem.hpp>
namespace fs = ghc::filesystem;

int main() {
    fs::path p = "/tmp/example/dir";
    fs::create_directories(p);
    for (const auto& entry : fs::directory_iterator(p.parent_path())) {
        // std::string here is interpreted as UTF-8, on every platform
        std::printf("%s\n", entry.path().string().c_str());
    }
    return 0;
}
```

To transparently prefer the real `std::filesystem` where it exists and fall back to `ghc` where it doesn't, include the helper header instead:

```cpp
#include <ghc/fs_std.hpp>   // defines `fs` as std::filesystem or ghc::filesystem
```

`fs_std.hpp` performs the `__has_include(<filesystem>)` probe (plus the Apple deployment-target checks) so you don't hand-write the preprocessor dance yourself.

## Architecture / How It Works

The core is a single header implementing `path`, the directory iterators, the free functions (`copy`, `remove_all`, `status`, `permissions`, etc.), the error-code and throwing overload pairs, and `fstream` wrappers (`ghc::filesystem::ifstream`/`ofstream`/`fstream`) that accept `path` on compilers whose own `fstream` predates that support. It is closely based on chapter 30.10 of the C++17 standard (working draft N4687); when compiled as C++20 it adapts to N4860 — the `std::u8string` handling and the changed `path` comparison/sorting order[^1].

The project ships three consumption models, and the choice matters more than it first appears:

1. **Single-file header** (`ghc/filesystem.hpp`) — simplest, but *not* namespace-clean. Because it is header-only it drags system includes into every translation unit that includes it; on Windows that means `Windows.h` leaks into your global namespace, so you are advised to define `WIN32_LEAN_AND_MEAN` / `NOMINMAX` before including it[^3].
2. **Forwarding + implementation split** (`ghc/fs_fwd.hpp` + `ghc/fs_impl.hpp`) — added in v1.1.0. Most translation units include the lightweight forwarding header; exactly one `.cpp` includes the implementation header. This confines the system includes (and Windows.h pollution) to that single file and cuts compile time across large builds. The `fs_std_fwd.hpp` / `fs_std_impl.hpp` pair layers the std-or-ghc auto-selection on top.
3. **Submodule + CMake** — `add_subdirectory()` then link the interface target.

A hard architectural constraint: the implementation cannot be hidden behind a Windows DLL boundary. A DLL interface exposing C++ standard templates is, in the author's words, "a different beast," and is explicitly unsupported. Earlier versions (through v1.4.0) implemented directory traversal on top of higher-level C/POSIX calls; the v1.5.x line moved toward a more native per-platform backend, which is why v1.4.0 is called out as the "pre-native-backend" release[^4].

## Production Notes

- **UTF-8 semantics are contagious and load-bearing.** In a codebase that mixes `ghc::filesystem` and `std::filesystem` (e.g. via `fs_std.hpp` on a machine where std is available), the *same* `path("...").string()` can round-trip bytes differently, because `ghc` guarantees UTF-8 and `std` uses an implementation-defined narrow encoding (often the active code page on Windows). Pick one behavior per data path and test on Windows specifically.
- **Windows symlinks and permissions are the documented weak spots.** Creating symbolic links on Windows requires the privilege (or Developer Mode); the test suite emits informational warnings rather than failing when it can't. Permission bits do not map cleanly to the POSIX model, so `permissions()` / `status().permissions()` behavior differs from Linux/macOS[^3].
- **Compile-time cost is real for the header-only mode.** Every TU that includes the single-file header re-parses the full implementation plus `Windows.h`. On non-trivial Windows projects the forwarding/implementation split is not a nicety but the recommended default.
- **Only the latest minor release gets bugfixes.** The author states plainly that fixes land on the newest release line; older lines (v1.3.x pre-C++20, v1.4.x pre-native-backend) are pinned snapshots, not maintained branches. Upgrading forward is expected[^4].
- **Not all "supported" platforms are CI-tested.** macOS, Windows, several Linux distros, FreeBSD, and (since v1.5.12) Solaris run in CI. Android NDK, Emscripten, QNX, GNU/Hurd, and Haiku are reported to work but are not automatically tested — treat them as best-effort.
- **Coverage is high (>90%) and it builds a conformance cross-check.** When the host compiler is recent enough, the test build also compiles the suite against the real `std::filesystem` (`std_filesystem_test`), so divergences between `ghc` and the platform standard library surface during development rather than in production.

## When to Use / When Not

**Use when:**
- You must support C++11 or C++14, or a platform/OS-version where `<filesystem>` is missing (old Apple deployment targets, older embedded/console toolchains).
- You want consistent UTF-8 path handling across Windows and POSIX without hand-rolling wide-string conversions.
- You need a vendored, dependency-free single header you can drop into a build with no extra link flags (`-lstdc++fs` / `-lc++fs`).

**Avoid when:**
- Your minimum toolchain already ships a complete, linkable `<filesystem>` — use the standard library directly; this backport is dead weight then.
- You require byte-for-byte conformance with `std::filesystem`'s narrow-string encoding on Windows (the UTF-8 choice is intentional and non-negotiable here).
- You need the implementation behind a Windows DLL boundary — unsupported.

## Alternatives

- std::filesystem — the standard library itself; the correct choice on any C++17+ toolchain that ships it. `gulrak/filesystem` is a fallback for when this isn't usable.
- boostorg/filesystem — the mature predecessor `std::filesystem` was standardized from; broader history and features, but a compiled (non-header-only) Boost dependency rather than a single header.
- Use `ghc/fs_std.hpp` over either when you want one source tree to auto-select std where present and the backport only where absent.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.x | 2018 | Initial public releases; header-only C++11–17 `std::filesystem` backport[^1]. |
| 1.1.0 | 2019 | Forwarding/implementation header split (`fs_fwd.hpp` / `fs_impl.hpp`); submodule + CMake support. |
| 1.3.10 | — | Last pre-C++20-support release line[^4]. |
| 1.4.0 | — | C++20 adaptation (N4860: `u8string`, path sorting order); last "pre-native-backend" release[^4]. |
| 1.5.x | — | Native per-platform backend; Solaris CI (1.5.12); QNX (1.5.6), GNU/Hurd + Haiku (1.5.14) reported working. |
| 1.5.14 | — | Latest release as of this writing[^4]. |

## References

[^1]: gulrak/filesystem README — implementation scope and standard basis (C++17 draft N4687, C++20 draft N4860). https://github.com/gulrak/filesystem
[^2]: README, "Why the namespace GHC?" — `ghc` = "gulrak's helper classes," unrelated to Haskell. https://github.com/gulrak/filesystem#why-the-namespace-ghc
[^3]: README, "Platforms" / "Differences" — UTF-8 Everywhere philosophy, Windows.h pollution, symlink and permission caveats. https://github.com/gulrak/filesystem
[^4]: README, "Downloads" / release notes — v1.3.10 (pre-C++20), v1.4.0 (pre-native-backend), v1.5.x, latest v1.5.14; only latest minor line receives bugfixes. https://github.com/gulrak/filesystem/releases

## Tags

cpp, c-plus-plus, filesystem, header-only, cpp11, cpp17, std-filesystem, cross-platform, utf-8, backport
