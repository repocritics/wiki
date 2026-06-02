# kubernetes/kubernetes

The production-grade container orchestrator. The de facto compute substrate for cloud-native deployments since the mid-2010s.

## What it is

A Go-based orchestrator that schedules containerized workloads across a cluster of nodes, manages their lifecycle, handles networking and storage, and exposes a declarative API for desired-state configuration. Originated at Google (descended from Borg) and donated to the Cloud Native Computing Foundation (CNCF). The reference implementation. Major cloud providers ship managed variants (EKS, GKE, AKS, OKE); self-hosted distributions include kubeadm, k3s (lightweight), and OpenShift (enterprise).

## Key features

- Declarative desired-state API — describe what you want, controllers reconcile to match.
- Resource types: Pod, Deployment, Service, Ingress, StatefulSet, DaemonSet, Job, CronJob, plus CRDs for custom extensions.
- Scheduler matches workloads to nodes by resource requests, affinity rules, taints/tolerations.
- Built-in horizontal autoscaling (HPA), vertical autoscaling (VPA), cluster autoscaling.
- Operator pattern — application-specific controllers extend the API.
- Pluggable runtimes (containerd, CRI-O), networking (CNI plugins), storage (CSI plugins).
- Apache 2.0 licensed.

## Tech stack

- Go primary across all core components (apiserver, controller-manager, scheduler, kubelet, kube-proxy).
- etcd as the state store.
- gRPC + Protobuf for internal component communication.
- OCI container images via containerd (default runtime).

## When to reach for it

- You're operating a distributed system that needs to schedule many containers across multiple machines.
- You want vendor-portable deployment — Kubernetes runs the same way on AWS / GCP / Azure / on-prem.
- You need a standard substrate for stateless services + a workable story for stateful workloads.

## When *not* to reach for it

- Your app fits on one VM or one Heroku-style PaaS — Kubernetes adds ~3 layers of complexity you don't need.
- You're a team of <5 — the operational tax of running Kubernetes outweighs the benefits.
- You're sensitive to operational complexity — Kubernetes is the canonical example of "powerful but with steep operational depth".

## Maturity signal

122k stars, 43k forks, Apache 2.0, last push the day this page was generated. 12-year-old CNCF Graduated project. The 2,732 open-issues count is moderate for a project of this scope and surface area. Release cadence is quarterly with N-2 version support; the maintenance and conformance ecosystem is the largest in open-source infrastructure.

## Alternatives

- HashiCorp Nomad — use when you want simpler orchestration with multi-runtime support (containers, VMs, binaries).
- Docker Swarm — use when you want Docker-native clustering without Kubernetes complexity (note: maintenance pace slowed).
- Apache Mesos — historically a competitor; effectively retired as Kubernetes consolidated the market.
- Managed PaaS (Heroku, Render, Fly.io, Railway) — use when you want zero-orchestration with platform constraints.

## Notes

The "complexity tax" critique is real but mostly relevant for small teams. The CNCF stewardship + Apache 2.0 license + vendor-portability make Kubernetes the safest substrate for large multi-team organizations. The "should we use K8s?" decision is rarely about technology — it's about whether your team can sustain the operational depth required. Managed offerings (EKS/GKE/AKS) absorb ~70% of that operational burden in exchange for vendor coupling.

## Tags

kubernetes, golang, containers, orchestration, cloud-native, cncf, devops, distributed-systems, apache-license, infrastructure
