# mirage/mirage

> A library operating system that compiles OCaml applications into single-purpose
> unikernels — the OS as a set of libraries you link, not a platform you run on.

[GitHub repo](https://github.com/mirage/mirage) ·
[Official website](https://mirageos.org) ·
License: ISC

## Overview

MirageOS is the original "library operating system" for the cloud era. Instead
of deploying an application onto a general-purpose OS, you write it in OCaml
against abstract device signatures (network, storage, clock, entropy), and the
`mirage` tool links only the libraries it actually needs into a single
standalone kernel image that boots directly on Xen, KVM, BHyve, OpenBSD VMM,
or Muen — no processes, no users, no shell, memory footprint in the tens of
megabytes. The project started at the University of Cambridge in 2009; its
2013 ASPLOS paper introduced "unikernel" as the field's term of art[^1].

The defining tradeoff is radical minimalism at the price of ecosystem
exclusivity. Because everything above the hypervisor is OCaml, the community
has had to rewrite the world — TCP/IP (mirage-tcpip), TLS (ocaml-tls), DNS,
git (ocaml-git/Irmin) all exist as pure OCaml libraries; what has not been
rewritten is simply unavailable. There is no "apt install" escape hatch, which
makes MirageOS exceptional for small security-critical network services and a
poor fit for anything needing an existing Linux-shaped dependency.

At ~3.0k stars after 17 years, this is a niche project with outsized
influence: MirageOS networking libraries sit inside Docker Desktop's
vpnkit[^4]. Maintenance is active — v4.11.1 shipped July 2026[^2].

## Getting Started

```bash
opam install mirage        # needs OCaml >= 4.13, opam >= 2.1
git clone https://github.com/mirage/mirage-skeleton
cd mirage-skeleton/tutorial/hello
mirage configure -t hvt    # targets: unix, xen, qubes, hvt, spt, virtio, muen
make depend
dune build
solo5-hvt dist/hello.hvt   # boots the unikernel under KVM
```

A unikernel is a functor over abstract devices plus a `config.ml` that wires
concrete implementations at configure time:

```ocaml
(* unikernel.ml *)
module Hello (Time : Mirage_time.S) = struct
  let start _time =
    Logs.info (fun f -> f "hello, world");
    Lwt.return_unit
end
```

```ocaml
(* config.ml *)
open Mirage

let main = main "Unikernel.Hello" (time @-> job)
let () = register "hello" [ main $ default_time ]
```

## Architecture / How It Works

The repository is the `mirage` CLI, not the OS itself — the OS is the
constellation of `mirage-*` libraries it assembles. The pipeline:

1. **Configuration DSL (functoria).** `config.ml` describes the application as
   functor applications over device signatures (`Mirage_net.S`,
   `Mirage_kv.RO`, ...). This is ordinary OCaml, so device wiring is
   type-checked at configure time, not discovered at boot.
2. **Target selection.** `mirage configure -t <target>` picks concrete
   implementations per backend: Unix sockets for `-t unix` (development mode,
   an ordinary binary), the pure-OCaml mirage-tcpip stack over a Xen netfront
   or Solo5 network device for real targets.
3. **Build.** Since MirageOS 4 (March 2022)[^2], configuration emits a dune
   workspace and uses opam-monorepo to vendor all dependency sources,
   cross-compiled with an ocaml-solo5 toolchain — reproducible, at the cost of
   a heavier, solver-driven fetch step.
4. **Base layer.** Non-Xen targets run on Solo5[^3], a minimal sandboxed
   execution environment: `hvt` (KVM), `spt` (a seccomp-sandboxed Linux
   process — no hypervisor at all), `virtio`, Muen, and Genode. The kernel is
   a single address space; concurrency is cooperative Lwt, not threads.

There is no syscall boundary or privilege transition inside the image — the
attack surface is the hypervisor interface plus your memory-safe OCaml code.

## Production Notes

- **Single-core, cooperative.** One unikernel = one vCPU; a blocked Lwt
  promise stalls everything, with no preemption. Scaling means more instances
  behind a balancer, not more cores.
- **Library-solver pain is the real upgrade tax.** The `mirage-*` ecosystem is
  hundreds of small opam packages; major tool releases (3.x → 4.0 especially)
  changed the `config.ml` DSL and device signatures, requiring source ports.
- **Debugging is not Linux debugging.** No shell, no strace, no ssh — just
  structured logs and a gdb stub in solo5-hvt. The standard workflow is
  developing under `-t unix` with ordinary tooling and only targeting hvt/xen
  for deployment.
- **Deployment story is bring-your-own.** The output is a kernel image, not a
  container — no Kubernetes-native path; Robur's `albatross` is the closest
  thing to an orchestrator[^6]. Boot is millisecond-scale, which enabled
  research on booting unikernels on-demand per incoming request[^7].
- **It is actually used.** qubes-mirage-firewall replaces Qubes OS's
  Linux-based firewall VM in tens of MB of RAM instead of hundreds[^5]; Robur
  ships production DNS, VPN, and CalDAV unikernels[^6]; Nitrokey's NetHSM
  appliance runs MirageOS over Muen. All small, security-first network
  services — the honest shape of the niche.

## When to Use / When Not

**Use when:**
- You are building a small, long-lived network service (DNS, firewall, TLS
  endpoint, VPN) where attack surface and footprint matter more than features.
- You want a memory-safe stack from application code down to the hypervisor
  boundary, with the OS eliminated as an attack class.
- You already write OCaml, or the service is small enough to rewrite.

**Avoid when:**
- You need any existing non-OCaml dependency — databases, ML runtimes, media
  libraries. There is no compatibility layer by design.
- You want to run unmodified applications as unikernels (see alternatives).
- You need multi-core scaling inside one instance, or your team's operational
  muscle memory is containers + kubectl.

## Alternatives

- unikraft/unikraft — POSIX-compatible unikernel toolkit in C; use it to build
  unikernels from existing apps (nginx, Redis) without rewriting in OCaml.
- nanovms/nanos — runs unmodified Linux ELF binaries as unikernels; the
  pragmatic path when you want the isolation without any porting.
- cloudius-systems/osv — single-app kernel with a Linux ABI, historically
  strong for JVM workloads; development has slowed markedly.
- hermit-os/kernel — Rust library OS; the closest ideological cousin if your
  language is Rust rather than OCaml.
- firecracker-microvm/firecracker — not a unikernel: minimal microVMs running
  full Linux. Choose it for strong isolation with zero porting cost.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2009-12 | Repo created; Cambridge research project. |
| 1.0 | 2013-12 | First stable release, Xen-first; follows the ASPLOS unikernel paper[^1]. |
| 2.0 | 2014-07 | ARM support, Irmin storage, vchan inter-VM transport. |
| 3.0 | 2017-02 | Solo5 targets (KVM hvt and later seccomp spt) alongside Xen[^3]. |
| 4.0 | 2022-03 | dune + opam-monorepo build, systematic cross-compilation[^2]. |
| 4.11.1 | 2026-07 | Current release[^2]. |

## References

[^1]: Madhavapeddy et al., "Unikernels: Library Operating Systems for the Cloud" — ASPLOS 2013. https://anil.recoil.org/papers/2013-asplos-mirage.pdf
[^2]: mirage/mirage GitHub releases (version dates). https://github.com/mirage/mirage/releases
[^3]: Solo5 — sandboxed execution environment for unikernels. https://github.com/Solo5/solo5
[^4]: moby/vpnkit — Docker Desktop networking built from MirageOS libraries. https://github.com/moby/vpnkit
[^5]: qubes-mirage-firewall — MirageOS firewall VM for Qubes OS. https://github.com/mirage/qubes-mirage-firewall
[^6]: Robur cooperative — production MirageOS unikernels and the albatross orchestrator. https://robur.coop
[^7]: Madhavapeddy et al., "Jitsu: Just-In-Time Summoning of Unikernels" — NSDI 2015. https://www.usenix.org/conference/nsdi15/technical-sessions/presentation/madhavapeddy

## Tags

ocaml, unikernel, library-os, xen, kvm, solo5, cloud, security, virtualization, systems-programming, networking
