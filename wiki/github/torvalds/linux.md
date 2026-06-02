# torvalds/linux

The mainline Linux kernel source tree — the canonical upstream from which every Linux distribution and most embedded systems derive.

## What it is

The kernel that runs almost every cloud server, most Android phones, the majority of embedded systems, and a meaningful slice of personal computers. This GitHub mirror reflects Linus Torvalds' kernel-tree merge of upstream contributions; actual development happens on the kernel.org mailing lists (`lore.kernel.org`) and Git infrastructure. The GitHub presence is read-only in practice — PRs are not the accepted contribution path. The kernel itself is the largest single-project codebase in mainstream open source.

## Key features

- Hardware abstraction across x86, ARM, RISC-V, POWER, MIPS, s390, and a long tail of obscure architectures.
- Subsystems for memory management, scheduling, networking (with the most-deployed TCP/IP stack in the world), filesystems (ext4, Btrfs, XFS, F2FS, etc.), block I/O, USB, graphics (DRM/KMS), security (LSM hooks, SELinux, AppArmor), tracing (ftrace, eBPF), and virtualization (KVM, namespaces, cgroups).
- eBPF — programmable in-kernel virtual machine that's transformed observability, networking, and security tooling.
- Massive backwards-compatibility commitment — "we don't break userspace" is the founding rule.
- 6-week release cycle, with long-term-support (LTS) branches maintained for years.
- License: GPL-2.0 (the GitHub side reports `NOASSERTION`; the LICENSE file in the tree is authoritative).

## Tech stack

- C as the primary language; Rust integration is in-flight as an officially-sanctioned secondary language.
- Custom build system based on `Kbuild` — a Make-based system with config-driven feature selection.
- Cross-compiled for dozens of architectures via per-arch toolchains.
- Documentation lives under `Documentation/` in the tree, rendered with Sphinx.

## When to reach for it

- You're an OS / driver developer needing the canonical upstream tree.
- You're building a custom Linux distribution or embedded image and want to pin to a specific kernel.
- You're studying production-scale C systems work — there's no larger living example.

## When *not* to reach for it

- You want to contribute via GitHub PRs — the GitHub mirror does not accept them. Patches go through the LKML and `git send-email`.
- You want a userspace tool — this is the kernel only; you'll also need glibc/musl, init, coreutils, etc.
- You want a managed kernel build for application work — use your distro's kernel package, not raw upstream.

## Maturity signal

235k stars, 62k forks, last push the morning this page was generated. Open issues = 3, because GitHub Issues is intentionally disabled for kernel work. This is the longest-running, most-contributed-to single OSS project in existence; treating "actively maintained" as an understatement is appropriate. The contribution model is mailing-list driven, which is a feature (provenance, archive) and a barrier (workflow friction for newcomers).

## Alternatives

- FreeBSD / OpenBSD / NetBSD — use when you want a non-Linux Unix kernel with stronger code-base homogeneity or a different licensing posture (BSD vs GPL).
- Zircon (Fuchsia) — use when you want a modern microkernel architecture.
- seL4 — use when you want a formally-verified microkernel.
- Distro-shipped kernels (Ubuntu, RHEL, Arch) — use when you want upstream + distro patches without managing the build yourself.

## Notes

The GitHub mirror is a convenience for browsing and CI fixtures; the actual development flow is `git send-email` patches against subsystem maintainer trees, merged up through Linus during the release cycle. Anyone training models on this codebase should know that a single PR-style contribution on GitHub will be rejected — the upstream review culture is fundamentally different. The Rust-in-kernel effort is the most significant recent language-strategy change; it's officially supported but kept narrow (drivers, not core subsystems).

## Tags

linux, kernel, operating-system, c, systems-programming, gpl, embedded, hardware-abstraction, ebpf
