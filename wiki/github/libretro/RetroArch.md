# libretro/RetroArch

> The reference frontend for the libretro API — one portable C shell that runs hundreds of emulators and game engines as swappable "cores".

[GitHub repo](https://github.com/libretro/RetroArch) ·
[Official website](https://www.libretro.com) ·
[License: GPL-3.0](https://github.com/libretro/RetroArch/blob/master/COPYING)

## Overview

RetroArch is not an emulator. It is a frontend for **libretro**, a C API that reduces an emulator (or game engine, or media player) to a dynamic library exposing generic audio/video/input callbacks[^1]. RetroArch supplies everything around that boundary: video and audio output, input mapping, menus, savestates, shaders, netplay, recording. The emulators themselves — bsnes, MAME, DOSBox, mGBA, and hundreds more — ship separately as "cores". The project began in 2010 as SSNES, a frontend for bsnes's libsnes API; the API was generalized into libretro and the frontend renamed RetroArch in 2012.

The defining tradeoff is generality versus specificity. One binary, one config system, and one savestate/netplay/shader stack across every emulated system is operationally elegant — and it is why RetroArch runs on everything from Windows 95 to the Nintendo Switch to WebAssembly (the README lists 40+ platforms). The cost is that per-system nuance lives behind a famously dense settings tree, and each core's quirks (BIOS files, option naming, savestate stability) leak through the uniform surface. Standalone emulators frequently beat the libretro build of the same emulator on configurability and upstream freshness.

Development is active: 13.4k stars, ~2.2k forks, commits pushed daily as of mid-2026, with tagged releases a few times per year (v1.22.2 in November 2025). The ~3k open issues reflect the surface area of 40+ platforms times hundreds of cores more than neglect.

## Getting Started

```bash
# Linux (Flatpak — the maintained cross-distro channel)
flatpak install flathub org.libretro.RetroArch

# macOS
brew install --cask retroarch

# Windows: installer or portable zip from https://www.retroarch.com
```

Cores are installed from inside RetroArch (Main Menu → Online Updater → Core Downloader), then content is launched against a core. The CLI mirrors this:

```bash
# Run a game with an explicit core
retroarch -L ~/.config/retroarch/cores/mgba_libretro.so game.gba

# Verbose logging — the first thing support will ask for
retroarch -v -L mgba_libretro.so game.gba
```

The frontend is useless without at least one core, and cores are useless without content; for many consoles you also need BIOS files placed in the `system/` directory, which RetroArch does not and legally cannot provide.

## Architecture / How It Works

The contract is a single C header, `libretro.h`[^2]. A core implements `retro_load_game`, `retro_run` (one call per emulated frame), and `retro_serialize`/`retro_unserialize` (savestates), and emits frames, audio, and input polls through callbacks the frontend registers. Cores are built as plain shared libraries with no link-time dependency on RetroArch — the same `.so`/`.dll` loads into any libretro frontend.

Everything frontend-side is a **driver**: video (OpenGL 1/2/3, Vulkan, Direct3D 9–12, Metal, SDL, software), audio (ALSA, PulseAudio, PipeWire, CoreAudio, XAudio2, JACK, and more), input, and menu (RGUI is software-rendered and runs on a potato; XMB, Ozone, and MaterialUI are GPU-rendered). This driver matrix is how the same codebase spans consoles, phones, and 30-year-old PCs, and it is written in intentionally conservative, portable C.

Several signature features are built on the savestate primitive:

- **Rewind** keeps a ring buffer of delta-compressed states and steps backward in real time.
- **Run-ahead** (added in 1.7.2, 2018) runs the core ahead and rolls back via savestates each frame, removing frames of a core's internal input lag — at roughly doubled CPU cost.
- **Netplay** synchronizes peers with rollback over serialized states, GGPO-style; it only works with cores whose serialization is complete and deterministic.

A/V sync uses dynamic rate control: audio is resampled by fractions of a percent so audio and video clocks agree, instead of dropping frames. The multi-pass shader system compiles Slang shaders through SPIR-V for Vulkan, D3D, and Metal (GLSL for GL; Cg is deprecated), driving a large CRT-shader ecosystem maintained in sibling repos.

## Production Notes

- **Configuration sprawl is the tax.** `retroarch.cfg` is a flat file with thousands of keys, plus per-core, per-directory, and per-game overrides that layer on top. Overrides silently shadow the global config; "why does this setting not stick" is the perennial support question. Treat `config.def.h` defaults as the baseline and diff against it.
- **Cores are the actual risk surface.** Core versions, savestate formats, and option schemas change independently of the frontend. Savestates are not portable across core updates (in-game SRAM saves generally are). Pin core versions for anything you care about; netplay requires byte-identical core and content on both sides.
- **Nightlies vs releases.** The buildbot ships nightly builds of frontend and cores. Nightlies regress; stable tags are cut a few times a year. The buildbot infrastructure was hacked in August 2020 (no malicious binaries distributed, but nightly and core-update services were down for weeks)[^3] — mirror what you depend on.
- **Store builds are gutted.** The Google Play build lost its core downloader to Google's policy on downloading executable code; the Steam release (2021) distributes a curated core subset as DLC; the iOS App Store build (2024, after Apple's emulator policy change) is likewise constrained. Sideloaded / direct-download builds are the full product.
- **Latency tuning is manual.** Defaults favor compatibility. Real latency work means choosing an exclusive-fullscreen video driver, enabling hard GPU sync or frame delay, and possibly run-ahead — each trades CPU headroom or stability, and the right mix is per-machine and per-core.
- **GPL-3.0 covers the frontend; each core has its own license**, several non-commercial (Snes9x, some MAME derivatives). Anyone embedding RetroArch in a commercial device must audit per-core — bundled-emulator license violations on retro handhelds are a recurring ecosystem scandal.

## When to Use / When Not

**Use when:**
- You want one interface, one shader/netplay/savestate stack across many emulated systems, especially on TV/handheld/gamepad-first setups.
- You target unusual hardware — game consoles, single-board computers, the browser via Emscripten — where standalone emulators have no port.
- You need frontend features standalone emulators lack: run-ahead, rollback netplay, multi-pass CRT shaders, FFmpeg recording, RetroAchievements[^4].
- You are building an emulation appliance (Batocera-style) and want libretro as the stable integration boundary.

**Avoid when:**
- You emulate one system deeply — standalone bsnes/ares, Dolphin, DuckStation, or PCSX2 expose more per-system control and get upstream fixes first.
- You need a mouse-and-keyboard-native desktop app; the gamepad-centric menu frustrates users who expect conventional GUI conventions.
- You want out-of-the-box correctness without tuning; defaults are conservative and the settings tree is hostile to casual users.
- Commercial redistribution without legal review — the core license matrix is a minefield.

## Alternatives

- openemu/OpenEmu — macOS-native, curated UI on top of (partly libretro) emulator plugins; use it when you want a Mac app, not an appliance.
- mamedev/mame — the preservation-first standalone for arcade and much more; use it when accuracy and documentation trump frontend features.
- ares-emulator/ares — bsnes-lineage multi-system emulator focused on accuracy; use it for a cleaner standalone multi-system experience.
- TASEmulators/BizHawk — Windows-centric multi-system frontend built for tool-assisted speedruns; use it for frame-level scripting and TAS work.
- batocera-linux/batocera.linux — full Linux distro that embeds RetroArch; use it when you want the appliance, not the component.

## History

| Version | Date | Notes |
|---------|------|-------|
| SSNES | 2010-05 | First commit; frontend for bsnes's libsnes API. |
| Rename | 2012 | libsnes generalized to libretro; SSNES becomes RetroArch. |
| 1.0.0.0 | 2014-01 | First stable release across ~15 platforms. |
| 1.7.2 | 2018-04 | Run-ahead input latency reduction. |
| 1.9.x | 2021 | Steam release with cores as DLC. |
| 1.16.0 | 2023-09 | Steady multi-release-per-year cadence through the 1.1x line. |
| 1.19.0 | 2024-05 | Release train continues; iOS App Store return follows in 2024. |
| 1.20.0 | 2025-01 | Followed by 1.21.0 (2025-05) and the 1.22 line. |
| 1.22.2 | 2025-11 | Latest tagged release[^5]; master remains active into 2026. |

## References

[^1]: libretro documentation, "Libretro Overview". https://docs.libretro.com/development/libretro-overview/
[^2]: libretro API header, `libretro.h`. https://github.com/libretro/RetroArch/blob/master/libretro-common/include/libretro.h
[^3]: libretro blog, "The libretro infrastructure was hacked" — 2020-08. https://www.libretro.com/index.php/the-libretro-infrastructure-was-hacked-however-users-arent-at-risk/
[^4]: RetroAchievements — achievement system integrated in RetroArch. https://retroachievements.org/
[^5]: RetroArch releases. https://github.com/libretro/RetroArch/releases

## Tags

c, emulation, libretro, emulator-frontend, retro-gaming, cross-platform, vulkan, opengl, netplay, shaders, gaming
