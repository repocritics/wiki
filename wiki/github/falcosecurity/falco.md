# falcosecurity/falco

> Kernel-level runtime security for Linux — watches syscalls in real time and alerts when process behavior matches a rule.

[GitHub repo](https://github.com/falcosecurity/falco) ·
[Official website](https://falco.org) ·
[License: Apache-2.0](https://github.com/falcosecurity/falco/blob/master/COPYING)

## Overview

Falco is a runtime security agent that streams Linux kernel events — primarily syscalls, plus container and Kubernetes metadata — and evaluates them against a rules engine, emitting an alert the moment behavior matches[^1]. It answers a different question than image scanners or admission controllers: not "is this workload safe to run?" but "is this running workload doing something it shouldn't right now?" (a shell spawned in a container, a write to `/etc/`, an outbound connection from a database pod, a package manager launched in production).

It was created by Sysdig in 2016, donated to the CNCF in 2018, and became a graduated CNCF project in 2024[^2]. It is the de facto open-source reference for the "syscall detection" category and the engine underneath several commercial runtime-security products.

The defining tension is the **kernel instrumentation boundary**. Falco must observe every syscall on a busy host without dropping events and without destabilizing the kernel. That requirement drives most of its complexity: multiple driver implementations (kernel module, legacy eBPF, modern CO-RE eBPF), a per-CPU ring-buffer capture path, and a hard operational reality that under sufficient load Falco *will* drop events and you must tune for it. It is a detection tool, not an enforcement tool — by default it observes and alerts; it does not block the syscall it flagged.

## Getting Started

The common deployment is a DaemonSet on Kubernetes via the official Helm chart:

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=modern_ebpf
```

Rules are YAML. A condition uses the sysdig filter syntax (`evt.type`, `proc.name`, `fd.name`, `container.id`, …); `output` is the alert template:

```yaml
- rule: Terminal shell in container
  desc: A shell was spawned in a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
  output: >
    Shell in container (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
  priority: NOTICE
  tags: [container, shell, mitre_execution]
```

`macro` and `list` entries (e.g. `shell_procs`, `spawned_process`) are reusable predicates defined in the shipped ruleset; you compose your own rules from them.

## Architecture / How It Works

Most of Falco's code does not live in this repo. The binary here wires together **falcosecurity/libs** — `libscap` (event capture) and `libsinsp` (state tracking, filtering, container/k8s enrichment) — plus the rules engine and outputs[^3]. The drivers also live in `libs`.

The event path:

1. **Driver** captures syscalls in the kernel and writes them to per-CPU ring buffers. Three implementations exist: the **kernel module** (legacy, requires build/sign per kernel), the **legacy eBPF probe** (needs kernel headers to compile), and the **modern eBPF probe** (CO-RE — compiled once, portable across kernels, no per-host build)[^4]. Modern eBPF is the recommended default on kernels that support it (roughly 5.8+).
2. **libscap** pulls events from the buffers into userspace.
3. **libsinsp** maintains live state — a process table, open file descriptors, container metadata resolved from the container runtime, and Kubernetes metadata — and enriches each raw syscall so rules can reference high-level fields.
4. **Rules engine** evaluates conditions against the enriched event stream. Falco is single-threaded and stateful by design; the maintainers keep it C++ specifically because a GC'd, concurrent runtime doesn't fit a sequential, allocation-sensitive hot path[^5].
5. **Outputs** fan out: stdout/file/syslog, gRPC, or program output. In practice most operators run **falcosidekick** to route to Slack, SIEMs, S3, and other sinks.

The **plugins** framework (introduced 2022) decouples Falco from syscalls as the only event source[^6]. Source plugins feed non-syscall streams — Kubernetes audit logs, cloud audit trails (CloudTrail), GitHub events — through the same rules engine; extractor plugins add new filter fields. **falcoctl** is the companion CLI for installing and updating rules and plugins from OCI registries.

## Production Notes

**Dropped events are the central operational concern.** When syscall volume exceeds what userspace can drain, the ring buffers fill and the kernel drops events — meaning Falco can miss the exact event you deployed it to catch. Falco exposes drop metrics and a `syscall_event_drops` policy (alert, ignore, exit). Mitigations: raise buffer size (`syscall_buf_size_preset`), tune the modern-eBPF CPU set, and — most effective — **narrow which syscalls are captured** so the hot path carries less. A ruleset that references only a handful of syscalls should not be paying to capture all of them.

**Rule tuning is the real cost of ownership.** The default ruleset is deliberately noisy-on-purpose; run it unmodified in a busy cluster and you will get false positives (legitimate `kubectl exec`, CI runners spawning shells, package installs in build images). Expect an iterative allow-listing phase per environment. Rules are versioned and distributed separately (falcosecurity/rules) so they can update out of band from the binary.

**Driver/kernel coupling.** The kernel module and legacy eBPF probe must match the running kernel; kernel upgrades can break capture until a matching driver is fetched (driverkit/falcoctl automate this). Modern eBPF largely removes this class of pain but requires a recent kernel and BTF support. Managed/locked-down node images (some GKE/EKS/AKS variants, minimal distros) are the usual friction points — verify driver support before rollout.

**It detects, it does not block.** Falco is not an inline enforcement engine. Response/blocking is bolted on downstream (falco-talon, custom consumers of the gRPC/output stream, or an operator reacting to alerts). If you need in-kernel prevention, this is the wrong layer.

**Managed offerings differ from upstream.** Vendors ship Falco with proprietary rule feeds and managed pipelines. The OSS project is the engine and a community ruleset; parity with a commercial detection catalog is not a given.

## When to Use / When Not

**Use when:**
- You want real-time detection of anomalous runtime behavior on Linux hosts or Kubernetes nodes.
- You need syscall-level visibility (process exec, file access, network) with container/pod context attached.
- You want an open, auditable, CNCF-graduated detection standard rather than a black-box agent.
- You'll feed alerts into a SIEM/data lake and have capacity to tune rules.

**Avoid when:**
- You need to *block* malicious actions inline — Falco alerts, it does not prevent (see Tetragon/KubeArmor).
- Your workloads are Windows or serverless/managed environments where you can't run a node-level agent or load a kernel driver.
- You want zero-tuning turnkey detection — the default rules require environment-specific allow-listing.
- Your need is point-in-time posture/compliance querying rather than a continuous event stream (osquery fits better).

## Alternatives

- aquasecurity/tracee — eBPF runtime detection and forensics with a Go/Rego signature engine; use when you want event forensics and portability with a signatures-as-code model.
- cilium/tetragon — eBPF observability *and* in-kernel enforcement (can kill processes); use when you need prevention, not just alerting, especially alongside Cilium.
- kubearmor/KubeArmor — LSM-based (BPF-LSM/AppArmor) runtime enforcement of pod policy; use when the goal is blocking disallowed behavior in Kubernetes.
- osquery/osquery — SQL over endpoint state; use for point-in-time posture and fleet querying rather than streaming syscall detection.
- wazuh/wazuh — broad host IDS/XDR (FIM, log analysis, compliance); use when you want an all-in-one security platform rather than a focused syscall detector.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2016 | Initial release by Sysdig[^1]. |
| — | 2018-10 | Donated to CNCF (sandbox)[^2]. |
| — | 2020-01 | Accepted to CNCF incubation. |
| 0.31.0 | 2022 | Plugins framework introduced[^6]. |
| 0.34+ | 2023 | Modern CO-RE eBPF probe matures; falcoctl for driver/rule management[^4]. |
| — | 2024 | Graduated CNCF project[^2]. |

## References

[^1]: Falco README and project overview. https://github.com/falcosecurity/falco
[^2]: CNCF, "Falco project journey" (donation 2018, graduation 2024). https://www.cncf.io/projects/falco/
[^3]: Falco core libraries (libscap / libsinsp / drivers). https://github.com/falcosecurity/libs
[^4]: Falco drivers and the modern eBPF probe. https://falco.org/docs/concepts/event-sources/drivers/
[^5]: Falco README FAQ, "Why is Falco in C++ rather than Go?". https://github.com/falcosecurity/falco#faqs
[^6]: Falco plugins framework and registry. https://github.com/falcosecurity/plugins

## Tags

security, runtime-security, ebpf, kubernetes, cloud-native, cncf, c-plus-plus, container-security, syscall-monitoring, observability, linux, threat-detection
