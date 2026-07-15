# kata-containers/kata-containers

> Lightweight VMs that run OCI containers with hardware-enforced isolation, presenting themselves to Kubernetes as an ordinary container runtime.

[GitHub repo](https://github.com/kata-containers/kata-containers) ·
[Official website](https://katacontainers.io) ·
[License: Apache-2.0](https://github.com/kata-containers/kata-containers/blob/main/LICENSE)

## Overview

Kata Containers wraps each container (or Kubernetes pod) in its own lightweight
virtual machine with a dedicated guest kernel, then makes that VM behave like a
normal container to the layer above. It plugs into containerd or CRI-O as a
`containerd-shim-v2` runtime, so a Kubernetes `RuntimeClass` is all that stands
between a standard pod and one that is isolated by a hypervisor rather than by
shared-kernel namespaces and cgroups. The pitch is a hard security boundary
(a separate kernel per workload) without asking operators to abandon the OCI /
CRI / Kubernetes toolchain they already run.

The project was formed in December 2017 by merging Intel's Clear Containers and
Hyper.sh's runV, and was hosted first under the OpenStack Foundation, later the
Open Infrastructure Foundation[^1]. This repository holds the 2.0-and-newer code
line; the 1.x runtime lived in separate repos. Since the 3.x series the project
has gained a second, Rust runtime (`runtime-rs`) and become the substrate for
Confidential Containers, running workloads inside hardware TEEs[^2].

The defining tradeoff is honest and unavoidable: you buy a real isolation
boundary and pay for it in per-workload memory (each VM carries its own kernel
and agent), boot latency, and a more complex I/O path (virtio-fs, virtio-blk,
vsock) than a shared-kernel runtime like runc. Kata is a good fit precisely when
that price is worth paying — multi-tenant, untrusted, or regulated workloads —
and a poor one when it is not.

## Getting Started

Kata is normally installed as a container-runtime component, not built from
source. On a Kubernetes node with containerd, the packaged install plus a
`RuntimeClass` is the usual path[^3]:

```bash
# Confirm the host can run Kata (VT-x/AMD-V, KVM, nested virt if in a VM)
kata-runtime check --verbose
```

```yaml
# RuntimeClass registered by the Kata deploy (e.g. kata-qemu)
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-qemu
handler: kata-qemu
---
apiVersion: v1
kind: Pod
metadata:
  name: isolated
spec:
  runtimeClassName: kata-qemu   # this pod boots inside its own VM
  containers:
    - name: app
      image: nginx
```

Without `runtimeClassName`, the pod runs on the default runtime (runc); with it,
containerd dispatches to the Kata shim and the pod comes up inside a VM.

## Architecture / How It Works

A Kata "container" is a process tree running inside a guest VM. The pieces:

- **Shim / runtime** — `containerd-shim-kata-v2` implements the containerd
  Task/shim-v2 API. It launches and manages one VM per pod (the sandbox) and
  translates OCI lifecycle calls into agent commands. Two implementations
  coexist: the mature Go runtime (`src/runtime`) and the newer Rust runtime
  (`src/runtime-rs`).
- **Hypervisor** — the VMM that boots the sandbox. QEMU is the default and most
  featureful; Cloud Hypervisor and Firecracker target faster boot and smaller
  attack surface; **Dragonball** (`src/dragonball`) is a built-in rust-vmm-based
  VMM compiled into `runtime-rs` so no external hypervisor process is needed;
  StratoVirt and others are also supported on specific arches.
- **Guest kernel + rootfs** — a minimal kernel and a "mini-OS" initrd/rootfs
  produced by `osbuilder`, tuned to boot fast and small.
- **Agent** — a process (`src/agent`, Rust) running as PID-adjacent init inside
  the VM. It sets up the container namespaces, mounts, and cgroups *inside* the
  guest and runs the workload. The runtime talks to it over **ttRPC on a vsock**
  channel — no network is exposed for control.

Storage and networking cross the VM boundary through paravirtualized devices:
root filesystems are shared into the guest with **virtio-fs**, block volumes with
virtio-blk, and networking via tap/`virtio-net` plumbed to the CNI interfaces on
the host. This device path is where much of Kata's performance character — and
its footguns — live.

Confidential Containers builds on this: with Intel TDX, AMD SEV-SNP, or IBM
Secure Execution the guest memory is encrypted and attested, and the image is
pulled and decrypted inside the guest so the host operator never sees plaintext[^2].

## Production Notes

- **Memory accounting is per-VM.** Every pod carries a guest kernel plus agent;
  the baseline overhead (tens to ~130+ MB depending on config) is multiplied by
  pod count. Templating/VM factory and memory hotplug reduce but do not remove
  it. Capacity planning that assumes runc-level density will be wrong.
- **Nested virtualization.** Running Kata inside a VM (most cloud nodes) requires
  nested virt to be enabled, or a bare-metal node pool. `kata-runtime check` is
  the first thing to run when pods fail to start; the failure is usually "no KVM."
- **virtio-fs quirks.** Shared-filesystem semantics differ subtly from a bind
  mount: caching modes affect coherency, and workloads sensitive to inode or
  mmap behavior can misbehave. DAX and cache tuning matter for I/O-heavy pods.
- **Boot latency vs. features.** Firecracker/Cloud Hypervisor cut sandbox start
  toward the low hundreds of ms; QEMU is slower but supports more devices (GPU/
  device passthrough, hotplug). Pick the hypervisor per workload via distinct
  RuntimeClasses, not globally.
- **Two runtimes, uneven parity.** The Go runtime is the long-standing default;
  `runtime-rs` + Dragonball is where new work (and confidential computing on some
  paths) is heading. Feature and hypervisor support differ between them — verify
  your hypervisor is supported on the runtime you deploy.
- **Debuggability.** A crashing workload is now inside a VM; `kata-ctl`,
  `agent-ctl`, guest console logging, and trace-forwarder exist because normal
  host `ps`/tracing visibility stops at the VM boundary.

## When to Use / When Not

**Use when:**
- You run multi-tenant or untrusted workloads and shared-kernel isolation
  (namespaces + seccomp + AppArmor) is not a strong enough boundary.
- You need confidential computing — encrypted, attested guests via TDX/SEV-SNP.
- You want VM-grade isolation while keeping the OCI/CRI/Kubernetes toolchain.

**Avoid when:**
- Density and cost per pod dominate: the per-VM memory and boot cost is real.
- Your nodes can't provide KVM / nested virtualization.
- Workloads depend on host kernel features, exotic bind-mount semantics, or
  devices the chosen hypervisor doesn't expose.
- Shared-kernel hardening (gVisor's syscall interception, or plain runc + strong
  policy) already meets your threat model at lower overhead.

## Alternatives

- google/gvisor — user-space kernel that intercepts syscalls; isolation without a
  full VM. Use instead when you want a lighter boundary and can accept syscall
  compatibility gaps.
- firecracker-microvm/firecracker — the microVM monitor itself (also a Kata
  hypervisor backend). Use directly when you're building your own VM-per-workload
  system rather than plugging into Kubernetes RuntimeClass.
- opencontainers/runc — the default shared-kernel OCI runtime. Use when workloads
  are trusted and you want maximum density and minimum overhead.
- containers/crun — faster C reimplementation of runc; same shared-kernel model.
  Use when you want runc semantics with lower per-container cost.
- confidential-containers/confidential-containers — the CoCo project built on
  Kata; engage there when confidential computing is the primary goal.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017-12 | Project formed by merging Intel Clear Containers + Hyper.sh runV[^1]. |
| 1.0 | 2018-05 | First stable release of the merged runtime[^1]. |
| 2.0 | 2020 | Rewrite; containerd shim-v2 runtime, Rust agent. This repo's baseline. |
| 3.0 | 2022 | `runtime-rs` (Rust runtime) and Dragonball built-in VMM introduced[^2]. |
| 3.x | 2023–2026 | Confidential Containers integration, TEE attestation, ongoing. |

## References

[^1]: Kata Containers project history and governance — Open Infrastructure Foundation. https://katacontainers.io/learn/
[^2]: Kata Containers design docs (architecture, runtime-rs, Dragonball, confidential computing). https://github.com/kata-containers/kata-containers/tree/main/docs/design
[^3]: Kata Containers installation and Kubernetes integration guides. https://github.com/kata-containers/kata-containers/tree/main/docs/install

## Tags

rust, go, containers, virtualization, kubernetes, kvm, qemu, firecracker, oci, confidential-computing, security, cri
