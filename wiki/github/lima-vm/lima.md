# lima-vm/lima

> Linux VMs on macOS (and other hosts) with automatic file sharing and port forwarding — the plumbing under Colima, Rancher Desktop, and Finch.

[GitHub repo](https://github.com/lima-vm/lima) ·
[Official website](https://lima-vm.io/) ·
[License: Apache-2.0](https://github.com/lima-vm/lima/blob/master/LICENSE)

## Overview

Lima ("Linux Machines") launches Linux virtual machines with automatic file
sharing and SSH-based port forwarding, positioned as a WSL2-style experience for
macOS[^1]. It was created by Akihiro Suda, a containerd/BuildKit maintainer, and
its original goal was to bring containerd and nerdctl to Mac developers who
otherwise reached for Docker Desktop[^2]. It has since become a general-purpose
VM manager: it runs Docker, Podman, and Kubernetes equally well, and supports
non-macOS hosts (Linux, NetBSD).

Lima's real significance is as infrastructure rather than as an end-user tool.
Colima, Rancher Desktop, and AWS's Finch are all built on top of it, which means
a large fraction of "Docker Desktop replacement" setups on macOS are running
Lima underneath[^3]. It is a CNCF incubating project, which is unusual for a
desktop VM launcher and reflects that pedigree. As of 2026 the repository is
actively developed (frequent pushes, ~21k stars) with a substantial open-issue
backlog (~500), most of it feature requests and host/guest-specific edge cases
rather than critical breakage.

The defining tension is the backend split. On Apple Silicon, Lima can use either
QEMU (mature, portable, slow) or Apple's Virtualization.framework via `vz`
(fast, native, but with a narrower feature set and some correctness caveats).
Choosing between them — and the matching file-sharing mode — is the single
decision that most affects whether Lima feels fast or sluggish.

## Getting Started

```bash
brew install lima
limactl start                     # boots the default Ubuntu template
lima uname -a                     # run a command in the guest
```

```bash
# Docker with an automatically forwarded socket
limactl start template://docker
export DOCKER_HOST=$(limactl list docker --format 'unix://{{.Dir}}/sock/docker.sock')
docker run --rm hello-world
```

```yaml
# ~/my-vm.yaml — a minimal custom instance definition
vmType: "vz"                      # use Apple Virtualization.framework
images:
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-arm64.img"
    arch: "aarch64"
mounts:
  - location: "~"
    writable: false
  - location: "~/work"
    writable: true
```

```bash
limactl start ./my-vm.yaml
```

## Architecture / How It Works

A Lima instance is a hypervisor process plus a Linux guest configured from a YAML
template. The template declares the disk image (usually an official cloud image),
CPU/memory/disk sizing, mounts, port-forward rules, and `provision` scripts.
Guest first-boot configuration is handled through cloud-init, so provisioning is
declarative rather than a captured golden image[^4].

**VM backends.** On macOS, `vmType: qemu` runs a QEMU process; `vmType: vz` uses
Apple's Virtualization.framework (macOS 13+). `vz` is dramatically faster on
Apple Silicon and is the default for new instances on recent macOS, but it is a
thinner abstraction — some QEMU-only knobs do not apply, and running x86_64
guests under `vz` depends on Rosetta translation. QEMU remains the portable
fallback and the path for cross-architecture emulation.

**File sharing.** This is where most of the performance story lives. Lima
historically used reverse-sshfs (the host mounts into the guest over the SSH
connection), which is portable but slow for large I/O. It later added 9p, and
with `vz` it supports virtiofs, which is substantially faster. The mount type is
coupled to the VM backend, so switching backends usually means switching mount
semantics as well.

**Networking and port forwarding.** By default the guest uses user-mode
networking, and Lima watches the guest for listening ports and forwards them to
the host automatically over SSH — this is what makes `docker run -p` "just work"
without manual mapping. Shared/bridged networking with real IPs requires the
separate `socket_vmnet` helper, which needs elevated privileges to install[^5].

**Guest agent.** A small agent runs inside the VM to report events (such as newly
opened ports) back to `limactl` on the host, driving the dynamic port-forwarding
behavior. The host CLI, the SSH control connection, and this agent together form
the control plane; there is no long-running privileged daemon on the host in the
Docker Desktop sense.

Because Colima, Rancher Desktop, and Finch wrap Lima, their behavior and their
bugs frequently trace back to these same backend/mount/network choices.

## Production Notes

Lima is a developer-workstation tool, not server infrastructure; "production"
here means daily-driver reliability rather than fleet operations.

- **Backend choice dominates perceived speed.** On Apple Silicon, `qemu` guests
  are noticeably slower than `vz`. If Lima feels slow, the backend and the mount
  type are the first things to change, not the CPU/memory allocation.
- **File-share throughput is the classic footgun.** Bind-mounting a large source
  tree (node_modules, large Git repos, build caches) over reverse-sshfs can be
  painfully slow. Prefer virtiofs (`vz`), keep heavy build artifacts on the guest
  filesystem, and mount as few host paths as possible.
- **`vz` maturity caveats.** The `vz` backend is fast but younger than QEMU;
  historically it has had rough edges around nested virtualization, certain
  images, and some mount/networking corners. When something works under QEMU but
  not `vz`, that asymmetry is usually the cause.
- **Disk resizing is one-way in practice.** Growing an instance's disk is
  supported; shrinking is not, so oversizing `disk` up front is safer than
  planning to reclaim space later.
- **Shared networking requires privileged setup.** `socket_vmnet` must be
  installed with root-owned permissions; teams that need real per-VM IPs should
  budget for that install step and its upgrade friction.
- **Config lives in templates, and templates drift.** Instances are created from
  a template at `limactl start` time; editing the template later does not
  retroactively change running instances. Reproducibility comes from checking the
  YAML into version control, not from mutating a live VM.
- **Upgrades occasionally touch on-disk instance format.** Upgrading `limactl`
  while instances exist is generally smooth, but reading release notes before a
  major bump is worthwhile since instance metadata and defaults evolve.

## When to Use / When Not

**Use when:**
- You want containerd/nerdctl, Docker, Podman, or a throwaway Kubernetes on macOS
  without Docker Desktop's licensing or daemon model.
- You need a scriptable, YAML-declared, reproducible Linux VM with automatic file
  sharing and port forwarding.
- You are building a tool that needs a VM substrate on macOS and want to delegate
  the hypervisor/mount/network plumbing (the Colima/Rancher/Finch pattern).

**Avoid when:**
- You want a GUI desktop Linux VM with a display — Lima is headless and CLI-first;
  UTM fits that better.
- You need production/server virtualization or orchestrated VM fleets — this is a
  workstation tool, not a hypervisor platform.
- You need a fully managed, zero-config "click to get Docker" experience with a
  GUI — Colima or Rancher Desktop (both built on Lima) are the higher-level layer.

## Alternatives

- abiosoft/colima — built on Lima; use it when you want `docker`/`kubernetes` on
  macOS with near-zero config and don't need to touch the VM directly.
- rancher-sandbox/rancher-desktop — use when you want a GUI and a Kubernetes-first
  workflow; it wraps Lima on macOS.
- runfinch/finch — use when you want AWS's opinionated, Lima-backed container CLI
  with a supported vendor path.
- utmapp/UTM — use when you need a graphical, general-purpose QEMU/vz VM manager
  with a desktop display rather than a headless container host.
- canonical/multipass — use when you specifically want Canonical-managed Ubuntu
  VMs with a similar auto-networking model across macOS/Windows/Linux.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2021-06 | containerd/nerdctl on macOS via QEMU; reverse-sshfs mounts[^2]. |
| vz backend | 2022 | Apple Virtualization.framework support added alongside QEMU. |
| CNCF Sandbox | 2022 | Accepted into CNCF as a Sandbox project. |
| 1.0 | 2024-11 | First stable major release; project matured to 1.x line[^6]. |

## References

[^1]: Lima README — "Lima launches Linux virtual machines with automatic file sharing and port forwarding (similar to WSL2)." https://github.com/lima-vm/lima
[^2]: Lima documentation, project background and goals. https://lima-vm.io/docs/
[^3]: Lima README, "Adopters" — Rancher Desktop, Colima, Finch, Podman Desktop. https://github.com/lima-vm/lima#adopters
[^4]: Lima documentation, templates and cloud-init based provisioning. https://lima-vm.io/docs/config/
[^5]: Lima documentation, networking and socket_vmnet. https://lima-vm.io/docs/config/network/
[^6]: Lima releases. https://github.com/lima-vm/lima/releases

## Tags

go, virtual-machine, macos, containers, containerd, qemu, apple-silicon, developer-tools, cncf, docker-alternative, kubernetes
