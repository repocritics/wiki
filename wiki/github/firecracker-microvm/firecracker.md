# firecracker-microvm/firecracker

> A minimal Rust VMM on KVM that boots stripped-down Linux microVMs in ~125 ms — the isolation layer under AWS Lambda and Fargate.

[GitHub repo](https://github.com/firecracker-microvm/firecracker) ·
[Official website](https://firecracker-microvm.github.io) ·
[License: Apache-2.0](https://github.com/firecracker-microvm/firecracker/blob/main/LICENSE)

## Overview

Firecracker is a virtual machine monitor (VMM) written in Rust that uses the Linux Kernel Virtual Machine (KVM) to run lightweight guests called microVMs. It was built at Amazon Web Services to back the serverless execution of AWS Lambda and Fargate, and was open-sourced at re:Invent in November 2018[^1]. As of 2026 it has ~35k stars and remains actively developed (releases every two to three months, last push mid-2026), driven primarily by the AWS team with an external maintainer community.

The defining decision is subtraction. A general-purpose VMM like QEMU emulates a wide device tree — PCI, BIOS/UEFI, USB, SCSI, VGA, a legacy chipset — because it must boot arbitrary operating systems. Firecracker deletes almost all of that. It ships a handful of `virtio` devices (net, block, vsock, and later balloon/entropy/pmem), a serial console, a minimal `i8042` controller (used only to signal reset on x86_64), and nothing else. There is no BIOS: the guest kernel is loaded directly via the Linux boot protocol. The result is a per-microVM memory overhead of a few MiB and a boot-to-userspace time on the order of 125 ms, at the cost of only running purpose-built Linux (and, in practice, guests you control end to end)[^2].

The central tension is generality versus footprint. Firecracker is not a Docker replacement and not a desktop hypervisor — it is a hardware-isolation primitive that assumes an orchestrator sits above it (firecracker-containerd, Kata Containers, Flintlock, historically Weave Ignite) and a well-configured Linux host below it. Outside that shape, its missing features (no GPU passthrough, historically no PCI, Linux guests only, x86_64/aarch64 only) are dealbreakers rather than trades.

## Getting Started

Firecracker requires a Linux host with KVM (`/dev/kvm`) on x86_64 or aarch64. Download a release binary or build from source (the build runs inside a dev container):

```bash
git clone https://github.com/firecracker-microvm/firecracker
cd firecracker
tools/devtool build
# binary at: build/cargo_target/$(uname -m)-unknown-linux-musl/debug/firecracker
```

Firecracker is controlled entirely through a REST API served on a Unix socket. Start it, then configure and boot a microVM with a kernel image and a root filesystem:

```bash
firecracker --api-sock /tmp/fc.sock &

# point at an uncompressed kernel (vmlinux) and an ext4 rootfs image
curl --unix-socket /tmp/fc.sock -X PUT 'http://localhost/boot-source' \
  -d '{"kernel_image_path":"vmlinux","boot_args":"console=ttyS0 reboot=k panic=1 pci=off"}'

curl --unix-socket /tmp/fc.sock -X PUT 'http://localhost/drives/rootfs' \
  -d '{"drive_id":"rootfs","path_on_host":"rootfs.ext4","is_root_device":true,"is_read_only":false}'

curl --unix-socket /tmp/fc.sock -X PUT 'http://localhost/actions' \
  -d '{"action_type":"InstanceStart"}'
```

Defaults are deliberately small: 1 vCPU and 128 MiB of guest memory unless overridden via the `/machine-config` endpoint. For production, run under the bundled `jailer` binary, which applies a cgroup/namespace barrier and drops privileges before exec'ing Firecracker.

## Architecture / How It Works

**Single process, one API.** Each microVM is one Firecracker process exposing one HTTP API on one socket. There is no daemon managing many VMs; concurrency is achieved by running many processes. Guest vCPUs run on dedicated host threads via `KVM_RUN`; the VMM thread services device I/O and the API.

**Minimal device model.** Devices are `virtio` (MMIO transport, not PCI in the classic path) plus a serial UART and the reset controller. This is why boot is fast and the attack surface is small — there simply is not much guest-facing code. The `[Developer Preview]` PCI and device hot-plug/hot-unplug work in recent releases is the notable exception to the historically PCI-free design.

**Security layers, defense in depth.** Three barriers wrap the KVM boundary: (1) the guest is confined by hardware virtualization; (2) thread-specific `seccomp-bpf` filters restrict the Firecracker process to a small syscall allowlist, filtered per thread so the vCPU threads can do even less than the API thread; (3) the jailer establishes cgroup/namespace isolation and drops to an unprivileged user. AWS's security posture assumes all three plus a hardened host (`prod-host-setup.md`).

**rust-vmm.** Firecracker does not carry a monolithic device stack. Much of the low-level plumbing (KVM ioctl bindings, virtio queues, memory model) lives in the shared [rust-vmm](https://github.com/rust-vmm) crates, which Firecracker and cloud-hypervisor both consume. This is a deliberate ecosystem split: reusable VMM building blocks below, product-specific policy above.

**Snapshot/restore.** Firecracker can serialize a running microVM's full state (guest memory, vCPU registers, device state) to disk and resume it elsewhere. Combined with guest-memory copy-on-write and userfaultfd-backed demand paging, this underpins fast-start patterns (restore a pre-booted snapshot instead of cold-booting). It is powerful and also the sharpest footgun — see below.

## Production Notes

**Snapshot cloning breaks entropy and uniqueness.** Restoring the same snapshot into many microVMs clones all in-guest state, including the RNG seed, MAC addresses, and any secrets or nonces captured at snapshot time. The official guidance is explicit: resumed clones must re-seed `/dev/random` (via the virtio-rng/entropy device) and rotate identifiers, or you get cryptographic reuse across "fresh" VMs[^3]. This is not a bug; it is inherent to cloning and must be handled by the layer above.

**The host is part of the trust boundary.** Firecracker's isolation claims are contingent on the production host setup: specific kernel config, side-channel mitigations (the guidance covers SMT/hyperthreading policy, cache and TLB flush behavior, KVM settings), and the jailer. Running Firecracker on an unhardened host silently weakens the multi-tenant guarantees it advertises. Read `docs/prod-host-setup.md` before trusting it with untrusted guests.

**Bare metal, effectively.** Firecracker needs KVM, and nested virtualization is not a supported target. In practice you run it on bare-metal instances (AWS `*.metal`) or physical hosts. You cannot reliably run untrusted multi-tenant Firecracker inside another cloud VM.

**Linux guests only, and you build them.** There is no BIOS/UEFI and no broad device support, so you cannot boot arbitrary OS images. You supply an uncompressed `vmlinux` and a rootfs you assemble. Windows guests are out of scope.

**Tested surface is narrow and AWS-shaped.** The support matrix is specific AWS EC2 metal instances (Intel Cascade Lake through Granite Rapids, AMD Milan/Genoa, Graviton 2–4) with named host kernels (5.10, 6.1, 6.18). 8th-gen Intel is only supported on 6.1/6.18 due to Granite Rapids kernel support. Off-matrix hardware may work but is not covered by CI.

**Known gaps.** On aarch64 the `pl031` RTC device does not support interrupts, so RTC-alarm consumers (e.g. `hwclock`) fail. `InstanceStop` via the API is x86_64-only. Overcommit is real and encouraged (demand paging + CPU oversubscription are on by default), which means capacity planning is your job — oversubscribed hosts can thrash under correlated load.

**Upgrades.** The snapshot format has version constraints across Firecracker releases; cross-version snapshot restore is bounded by a documented compatibility policy, so a fleet mid-upgrade must account for which versions can load which snapshots. Treat snapshot compatibility as a first-class part of any rollout plan.

## When to Use / When Not

**Use when:**
- You are building a multi-tenant platform that runs many short-lived, untrusted Linux workloads and need hardware isolation without VM-scale overhead (serverless functions, CI runners, code sandboxes, per-request isolation).
- You want fast, dense start (thousands of microVMs per host, sub-second boot or snapshot-restore).
- You control the guest kernel and rootfs and can run on bare metal.
- You are integrating below an orchestrator (containerd, Kata, your own control plane), not shipping to end users directly.

**Avoid when:**
- You need to boot arbitrary OSes, GPUs/PCI passthrough, Windows, or a full device model — use QEMU.
- You cannot get bare metal / KVM (managed environments without nested virt).
- You want a container runtime or a developer-desktop VM — Firecracker is neither.
- Your isolation needs are satisfied by OS-level sandboxing and the operational cost of managing a VMM fleet is not justified — gVisor or plain containers may fit.

## Alternatives

- cloud-hypervisor/cloud-hypervisor — sibling rust-vmm-based VMM with a richer device model (PCI, device hotplug, more guest OSes); use it when you need more than Firecracker's minimal surface.
- qemu/qemu — the general-purpose VMM; use it when you need arbitrary OS/firmware/hardware emulation and can accept the footprint and attack surface.
- google/gvisor — userspace kernel sandbox (syscall interception, no KVM required); use it when you want container-shaped isolation without bare metal.
- kata-containers/kata-containers — OCI-compatible runtime that puts containers in microVMs and can drive Firecracker or QEMU underneath; use it when you want the Kubernetes/containerd integration done for you.
- crosvm (ChromeOS) — the Rust VMM Firecracker forked from; use it in the Chrome OS / crostini context.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-source launch | 2018-11 | Announced and open-sourced at AWS re:Invent[^1]. |
| 0.20 | 2020 | aarch64 support maturing; API and jailer stabilizing. |
| 0.23 | 2020 | Snapshot/restore introduced (beta)[^2]. |
| 1.0 | 2022-01 | First stable release; snapshot-restore, API stability commitments[^4]. |
| 1.x | 2022–2026 | Balloon, entropy, pmem, memory hotplug, CPU templates; PCI + device hotplug in developer preview. |

## References

[^1]: AWS News Blog, "Firecracker – Lightweight Virtualization for Serverless Computing" — 2018-11-26. https://aws.amazon.com/blogs/aws/firecracker-lightweight-virtualization-for-serverless-computing/
[^2]: Agache et al., "Firecracker: Lightweight Virtualization for Serverless Applications," USENIX NSDI 2020. https://www.usenix.org/conference/nsdi20/presentation/agache
[^3]: Firecracker snapshotting docs — snapshot support and safety (clone entropy/uniqueness caveats). https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md
[^4]: Firecracker CHANGELOG. https://github.com/firecracker-microvm/firecracker/blob/main/CHANGELOG.md

## Tags

rust, virtualization, microvm, vmm, kvm, serverless, sandbox, hypervisor, aws, container-isolation, security, linux
