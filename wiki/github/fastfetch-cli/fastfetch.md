# fastfetch-cli/fastfetch

> A neofetch-style system information tool written in C, built for speed and JSONC-driven customization.

[GitHub repo](https://github.com/fastfetch-cli/fastfetch) ·
[License: MIT](https://github.com/fastfetch-cli/fastfetch/blob/dev/LICENSE)

## Overview

Fastfetch prints a system-information summary next to an ASCII or image logo — OS,
kernel, host, CPU, GPU, memory, disk, uptime, terminal, and dozens of other
modules — the same category of tool as the once-ubiquitous neofetch. It was started
by Linus Dierheimer in 2021 and is now maintained primarily by @CarterLi
(zhangsongcui)[^1]. As of 2026 it has roughly 23.7k stars and ships releases on a
near-monthly cadence (2.66.0 landed 2026-07-10, and the repo was pushed to the day
before this page was written), so "actively maintained" is a factual claim here, not
a marketing one.

The reason fastfetch exists is that neofetch was a ~10,000-line Bash script that
gathered data by spawning subprocesses (`lspci`, `xrandr`, `df`, `cat /proc/...`) and
parsing their text output; it was slow, and it was archived by its author in 2024[^2].
Fastfetch reimplements the same idea in C, reading kernel and platform APIs directly
(`/sys`, `/proc`, `sysctl`, the Windows registry, macOS system frameworks) instead of
shelling out. The practical result is a startup-time tool fast enough to run on every
interactive shell launch without a noticeable delay — its main design constraint, since
that is the dominant use case.

It is genuinely cross-platform: Linux, macOS, Windows 8.1+, Android (Termux), the BSDs,
Haiku, and SunOS/illumos are supported, though the project only actively tests x86-64
and aarch64[^3]. The tradeoff for the C rewrite is that it is a compiled binary with
platform-specific detection code rather than a single portable script you can read and
patch in five minutes.

## Getting Started

Fastfetch is packaged nearly everywhere; prefer your package manager but check the
version, because distributions frequently ship outdated builds that receive no support[^3].

```bash
# Linux (examples)
sudo pacman -S fastfetch          # Arch
sudo dnf install fastfetch        # Fedora
sudo apt install fastfetch        # Debian 13+ / Ubuntu 25.04+

brew install fastfetch            # macOS / Linuxbrew
scoop install fastfetch           # Windows
pkg install fastfetch             # FreeBSD / Termux
```

Run it, then generate and edit a config:

```bash
fastfetch                         # default output
fastfetch -c all.jsonc            # every available module — a discovery tool
fastfetch --gen-config            # writes ~/.config/fastfetch/config.jsonc
fastfetch -s cpu:gpu:memory --format json   # structured output for scripting
```

A minimal `config.jsonc` (JSON with comments) showing per-module formatting:

```jsonc
{
    "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
    "modules": [
        "os",
        "kernel",
        { "type": "gpu", "format": "{name}" },   // fastfetch -h gpu-format
        "memory"
    ]
}
```

## Architecture / How It Works

Fastfetch is organized as a set of **modules**, one per line of output (OS, Host,
Kernel, CPU, GPU, Memory, Disk, Battery, Local IP, and so on). Each module has a
platform-specific detection backend under `src/detection/` and a presentation layer
that applies a `format` string. Detection is written against native APIs per OS, which
is why the codebase carries parallel implementations of, e.g., GPU enumeration for
Linux (DRM/`pci.ids`), Windows (WMI/DXGI), and macOS.

Configuration is **JSONC** validated against a published JSON schema[^4]. The same
schema generates the reference documentation, which the project itself admits is
machine-generated and not especially friendly. The intended editing workflow is an
IDE with schema support (VS Code) so completion and validation come from the schema.
Every module also exposes a `--<module>-format` flag with named placeholders, and
`--format json` emits the raw detected values as structured data — the feature that
makes fastfetch usable as a scripting primitive, not just a pretty banner.

The **logo system** is separate from data collection: logos can be built-in ASCII art
per distro, a custom text/ASCII file, or a real image rendered through a terminal
graphics protocol (sixel, kitty, iterm). Protocol support is a property of the
terminal, not fastfetch, which is the source of most "my image logo looks wrong"
reports. The build is CMake-based; optional features (Vulkan, OpenCL, image backends,
DRM, X11/Wayland) are compile-time flags, so a distro package's capabilities depend on
how it was configured.

## Production Notes

- **Packaged versions lag, and old versions are unsupported.** The maintainers
  explicitly decline to support outdated builds and change detection code frequently.
  If a module misbehaves, upgrading to the latest release is the first (and often only)
  supported fix — via the project's own `.deb`, Homebrew, or the nightly build before
  filing a bug[^3].
- **The `Command` module runs arbitrary shell.** Configs shared online can execute
  code on your machine. Fastfetch prints an explicit warning about this: review any
  config from an untrusted source before loading it. Treat `config.jsonc` as
  executable, not data.
- **`Local IP` is shown by default** and regularly surprises users. It exposes only
  private addresses (10./172./192.168.), which are not sensitive, but if you screenshot
  your terminal publicly you may want to disable the module.
- **Shell-startup gotchas.** With Powerlevel10k instant prompt, fastfetch must run
  *before* p10k initializes or output renders in black and white; the p10k contract
  forbids stdout after instant-prompt starts. `fastfetch --pipe false` forces color
  when output is detected as non-interactive.
- **GPU shows as `Device XXXX`** on Debian/Ubuntu when the system `pci.ids` database is
  stale; the fix is updating `pci.ids` (and `amdgpu.ids` for AMD) or passing
  `--gpu-driver-specific` to ask the driver directly.
- **Platform coverage is uneven.** Only x86-64 and aarch64 are actively tested; other
  architectures and the less-common OSes (Haiku, SunOS) work on a best-effort basis.
- Development happens on the **`dev` branch**, which is the default branch — clone and
  build from `dev`, not a nonexistent `main`/`master`.

## When to Use / When Not

**Use when:**
- You want a fast system-info banner on shell startup or in a screenshot/rice setup.
- You need it to work identically across Linux, macOS, Windows, and BSD.
- You want machine-readable system facts (`--format json`) for scripts or dashboards.
- You are migrating off neofetch, which is archived and no longer maintained.

**Avoid when:**
- You want a tiny, auditable, dependency-free script you can read end to end — a POSIX
  `sh` fetcher like pfetch fits better.
- You need a monitoring or inventory tool: fastfetch is a point-in-time display, not a
  metrics agent.
- Your platform is exotic (non-x86-64/aarch64) and you need guaranteed correctness.

## Alternatives

- dylanaraps/neofetch — the original Bash tool; archived and unmaintained since 2024. Use fastfetch instead for anything new.
- Macchina-CLI/macchina — Rust-based fetch tool, also fast; choose it when you prefer a Rust toolchain or its output style.
- dylanaraps/pfetch — minimal POSIX-sh fetcher; use when you want something tiny and readable over feature breadth.
- o2sh/onefetch — fetches *git repository* stats rather than system info; complementary, not a replacement.
- hykilpikonna/hyfetch — neofetch fork focused on pride-flag color themes; use for the theming, expect neofetch-era performance.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2021-02-18 | Repository created by Linus Dierheimer[^1]. |
| 1.0.0 | 2022-03-20 | First stable release. |
| 2.0.0 | 2023-08-14 | Major rewrite; new JSONC configuration and module system[^4]. |
| 2.66.0 | 2026-07-10 | Recent release; monthly-ish cadence with frequent patch releases. |

## References

[^1]: Fastfetch README — maintainers and project description. https://github.com/fastfetch-cli/fastfetch
[^2]: neofetch repository, archived by its author. https://github.com/dylanaraps/neofetch
[^3]: Fastfetch README, "Installation" and platform-support notes. https://github.com/fastfetch-cli/fastfetch#installation
[^4]: Fastfetch JSON schema and configuration wiki. https://github.com/fastfetch-cli/fastfetch/wiki/Configuration

## Tags

c, cli, system-information, neofetch, terminal, cross-platform, jsonc, command-line, ricing, developer-tools
