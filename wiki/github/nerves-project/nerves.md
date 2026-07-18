# nerves-project/nerves

> Embedded Linux reduced to a boot-loader for the Erlang VM — Elixir tooling
> that ships your whole device as a small, A/B-updatable firmware image.

[GitHub repo](https://github.com/nerves-project/nerves) ·
[Official website](https://nerves-project.org) ·
[License: Apache-2.0](https://github.com/nerves-project/nerves/tree/main/LICENSES)[^1]

## Overview

Nerves is a framework for building embedded systems in Elixir, started in 2016
by Justin Schneck and Frank Hunleth and stable since v1.0.0 in May 2018[^2]. Its
core idea inverts the usual embedded Linux stack: instead of a distribution
with an init system, shell userland, and package manager, Nerves boots a
minimal Buildroot-built Linux kernel whose PID-1 replacement (`erlinit`)
launches an Erlang/OTP release directly[^3]. Linux exists only to provide
drivers, networking, and a filesystem; everything above that — supervision,
logging, networking policy, the application — is BEAM code.

The payoff is that embedded developers get OTP's supervision trees, hot
introspection (IEx over SSH), and the entire Hex ecosystem — Phoenix LiveView
dashboards and even Livebook run on-device[^4]. The tradeoff is a hard
commitment to the BEAM: Nerves targets Linux-capable application processors
(Raspberry Pi, BeagleBone, STM32MP1, x86_64), not microcontrollers, and the VM
is soft-realtime only. This repository is specifically the build tooling — Mix
tasks, artifact resolution, cross-compilation glue — while the OS layers live
in sibling repos. At ~2.5k stars it is a niche-sized community, but an
unusually active one: v1.15.0 shipped 2026-07-13 with pushes the same week,
sustained by an Open Collective and long-lived core maintainers[^2].

## Getting Started

```bash
mix archive.install hex nerves_bootstrap
mix nerves.new hello_nerves
cd hello_nerves
export MIX_TARGET=rpi4        # or rpi0, bbb, x86_64, grisp2, ...
mix deps.get
mix firmware                  # cross-compile into a .fw image
mix burn                      # write to an SD card
```

After first boot, iterate over the network instead of reflashing:

```bash
mix firmware && mix upload nerves.local   # A/B update over SSH
ssh nerves.local                          # lands in a remote IEx shell
```

`MIX_TARGET` selects which prebuilt Nerves system (kernel + rootfs + toolchain)
is fetched; unset, the project compiles for the host so tests run normally.

## Architecture / How It Works

A Nerves firmware image is assembled from three layers:

1. **Nerves system** — a Buildroot-produced package per board
   (`nerves_system_rpi4`, etc.) containing the Linux kernel, a minimal
   squashfs root filesystem, and a matching cross-toolchain. Systems are
   ordinary Hex dependencies; precompiled artifacts are downloaded at
   `mix deps.get` time so users never run Buildroot for supported boards[^5].
2. **This repo's tooling** — Mix integration that resolves those artifacts,
   points NIF/port compilation (via `elixir_make`) at the cross-toolchain, and
   assembles the OTP release plus rootfs overlay into a firmware image.
3. **`fwup`** — a separate firmware packaging tool that produces the `.fw`
   archive and applies it using an A/B partition scheme: updates write to the
   inactive partition and a failed boot can fall back to the previous one[^6].

At runtime the boot chain is bootloader → Linux kernel → `erlinit` (PID 1) →
Erlang/OTP release[^3]. There is no systemd, no package manager, and by default
no shell utilities beyond a BusyBox subset. The root filesystem is a read-only
squashfs; application state goes to a writable `/data` partition. Networking
is handled in Elixir by `VintageNet`, logging by `RingLogger` (an in-memory
ring buffer — there is no journald), and introspection happens through IEx
over UART or SSH. Anything needed from the Linux world is enabled as a
Buildroot package in a custom system, never installed on a live device.

The coupling story: this repo is thin, but it pins tightly to Erlang/OTP
versions. The precompiled system artifact embeds a specific ERTS, and your host
toolchain's Elixir/OTP must be compatible with it — the framework checks and
refuses mismatches. Fleet OTA is delegated to NervesHub, a self-hostable
companion server maintained in a separate organization[^7].

## Production Notes

- **OTP/Elixir version pinning is the classic footgun.** Each system release
  ties to an ERTS version; upgrading host Elixir without bumping the system
  dependency (or vice versa) fails the build. Teams standardize on asdf/mise
  version files per project and upgrade host and target in lockstep.
- **Custom systems are the cost cliff.** Staying on official boards with
  default kernels is smooth. The moment you need a kernel config change,
  device-tree overlay, or extra Buildroot package, you fork a
  `nerves_system_*` repo and run Buildroot yourself — hours-long builds that
  require Linux (macOS users go through Docker or a VM)[^5].
- **Soft realtime only.** BEAM scheduling gives millisecond-class latency, not
  microsecond determinism. Production designs offload strict-timing I/O to a
  co-processor or a NIF; GPIO bit-banging from Elixir is not timing-safe.
- **Whole-image updates, not package updates.** Every change ships as a full
  firmware (base images run roughly 20–40 MB). This makes deployments
  reproducible and rollback trivial, but there is no `apt upgrade` escape
  hatch on a fielded device.
- **Read-only rootfs surprises ported code.** Libraries that assume a writable
  filesystem (caches, lock files) must be pointed at `/data`. Similarly, many
  boards lack an RTC — until NTP syncs, TLS certificate validation can fail,
  so time-dependent startup work needs a supervisor that tolerates it.
- **Bus factor.** Release cadence is healthy (roughly monthly through 2026, 22
  open issues), but the core team is a handful of people and per-board support
  depends on volunteers keeping each `nerves_system_*` current.

## When to Use / When Not

**Use when:**
- Your team knows Elixir/OTP and wants supervision trees, remote IEx, and
  LiveView UIs on a device instead of a C/embedded-Linux toolchain.
- The product runs on Linux-capable hardware (64 MB+ RAM class) and benefits
  from atomic A/B firmware updates with rollback.
- You want a locked-down appliance image — no shell, read-only rootfs — rather
  than a general-purpose Linux install to harden.

**Avoid when:**
- The target is a microcontroller: Nerves needs Linux; AtomVM covers the BEAM
  story on ESP32-class chips.
- You need hard realtime guarantees or minimal power budgets.
- The team has no BEAM experience and the product is a one-off — the Elixir
  learning curve plus the custom-system cliff outweighs the runtime benefits.
- You depend heavily on existing Linux userland services; re-enabling them
  through Buildroot one by one fights the framework's design.

## Alternatives

- buildroot/buildroot — build a conventional embedded Linux image directly
  when you want the minimal-OS approach without committing to the BEAM.
- yoctoproject/poky — layer-based industrial embedded Linux; use when vendor
  BSPs, long product lifecycles, and compliance processes dominate.
- balena-io/balena — container-based device fleet OS; use when the team wants
  Docker workflows and language-agnostic deployment over firmware images.
- atomvm/AtomVM — Erlang/Elixir on microcontrollers (ESP32, STM32); use below
  the Linux hardware floor where Nerves cannot go.
- micropython/micropython — high-level-language embedded development on MCUs
  when the goal is scripting convenience rather than OTP semantics.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.2.0 | 2016-06-27 | First tagged release of the Mix tooling[^8]. |
| v1.0.0 | 2018-05-01 | Stable release; tooling and system contract stabilized[^2]. |
| v1.7.0 | 2020-10-07 | Long-lived 1.7.x line (17 patch releases through 2022). |
| v1.9.0 | 2022-08-23 | Steady annual cadence resumes after 1.7.x maintenance era. |
| v1.11.0 | 2024-07-05 | 1.11.x line spans a year of maintenance releases. |
| v1.13.0 | 2026-01-21 | Start of a faster 2026 release cadence. |
| v1.15.0 | 2026-07-13 | Current release[^8]. |

## References

[^1]: The repo is REUSE-compliant: code under Apache-2.0, docs under
    CC-BY-4.0, with per-file SPDX headers — which is why GitHub's license
    detector shows no single license. https://github.com/nerves-project/nerves/tree/main/LICENSES
[^2]: Nerves project website and README. https://nerves-project.org
[^3]: erlinit — replacement for /sbin/init that launches an Erlang/OTP
    release. https://github.com/nerves-project/erlinit
[^4]: Nerves Livebook — Livebook running on Nerves devices.
    https://github.com/nerves-livebook/nerves_livebook
[^5]: nerves_system_br — Buildroot-based build platform for Nerves systems.
    https://github.com/nerves-project/nerves_system_br
[^6]: fwup — configurable embedded Linux firmware update creator and runner.
    https://github.com/fwup-home/fwup
[^7]: NervesHub — self-hostable OTA firmware update server.
    https://github.com/nerves-hub/nerves_hub_web
[^8]: GitHub releases, nerves-project/nerves.
    https://github.com/nerves-project/nerves/releases

## Tags

elixir, erlang, embedded, embedded-linux, firmware, iot, buildroot, raspberry-pi, ota-updates, beam
