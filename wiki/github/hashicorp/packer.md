# hashicorp/packer

> Builds identical machine images for many platforms from one source configuration — the canonical "golden image" tool for immutable infrastructure.

[GitHub repo](https://github.com/hashicorp/packer) ·
[Official website](https://developer.hashicorp.com/packer) ·
[License: BUSL-1.1](https://github.com/hashicorp/packer/blob/main/LICENSE)

## Overview

Packer is a CLI that automates the creation of machine images — AMIs, Azure managed
images, GCE images, VMware/VirtualBox/QEMU disks, Docker images, and dozens of
others — from a single declarative template[^1]. It was HashiCorp's second product
(Mitchell Hashimoto, first released June 2013), and it established the "golden
image" workflow that underpins immutable infrastructure: bake a fully-provisioned
image once, then deploy that artifact unchanged, rather than configuring servers
after boot.

The defining idea is a single source of truth that fans out to many targets in
parallel. One template describes what to install and configure; a set of *builders*
each spin up a throwaway machine on their respective platform, run the same
*provisioners* against it, and snapshot the result. This is why Packer pairs so
naturally with Terraform (which consumes the image) and Vagrant (which packages
images as dev boxes) — it produces the artifact the rest of the HashiCorp stack
deploys.

The central tension is that "identical" is a promise Packer orchestrates but does
not enforce. Because provisioning pulls from live sources (base AMIs, apt/yum
mirrors, upstream package versions), two runs of the same template days apart can
produce materially different images unless every input is pinned. Packer gives you
the machinery for reproducibility; reproducibility itself remains the operator's
job.

## Getting Started

```bash
# macOS (Homebrew)
brew tap hashicorp/tap && brew install hashicorp/tap/packer
# or download a binary from developer.hashicorp.com/packer/install
```

```hcl
# build.pkr.hcl — build an Ubuntu AMI on AWS
packer {
  required_plugins {
    amazon = {
      source  = "github.com/hashicorp/amazon"
      version = "~> 1.3"
    }
  }
}

source "amazon-ebs" "ubuntu" {
  ami_name      = "packer-ubuntu-{{timestamp}}"
  instance_type = "t3.micro"
  region        = "us-east-1"
  source_ami_filter {
    filters     = { name = "ubuntu/images/*ubuntu-jammy-22.04-amd64-server-*" }
    owners      = ["099720109477"]
    most_recent = true
  }
  ssh_username = "ubuntu"
}

build {
  sources = ["source.amazon-ebs.ubuntu"]
  provisioner "shell" {
    inline = ["sudo apt-get update", "sudo apt-get install -y nginx"]
  }
}
```

```bash
packer init build.pkr.hcl      # download declared plugins
packer fmt   build.pkr.hcl      # canonicalize formatting
packer validate build.pkr.hcl   # static checks
packer build   build.pkr.hcl    # spin up, provision, snapshot, tear down
```

## Architecture / How It Works

A Packer build is orchestrated by a small Go core that drives four component types,
all supplied by plugins[^2]:

- **Builders** create and manage the target machine and produce the final image
  (`amazon-ebs`, `azure-arm`, `googlecompute`, `vsphere-iso`, `qemu`, `docker`, …).
- **Provisioners** run against the live machine to install and configure software
  (`shell`, `file`, `ansible`, `powershell`, `chef`, `puppet`).
- **Post-processors** transform the resulting artifact (`vagrant`, `compress`,
  `docker-push`, `manifest`, `checksum`).
- **Communicators** are the transport Packer uses to reach the machine — SSH,
  WinRM, or `none`.

Since Packer 1.7 (2021) plugins are **separate binaries**, not bundled into the
core. The core talks to each plugin over gRPC using HashiCorp's `go-plugin`
library — the same model Terraform uses for providers. `packer init` reads the
`required_plugins` block and downloads matching releases from GitHub / the plugin
registry. This decoupling let HashiCorp shrink the core and let plugin authors ship
on their own cadence, at the cost of an extra install step and a version-matrix to
manage.

Templates come in two languages. The original was **JSON**; since 1.5 (2019) the
recommended language is **HCL2**, the same configuration language as Terraform,
which brings variables, functions, `for` expressions, and data sources. JSON
templates still run but are deprecated; `packer hcl2_upgrade` mechanically converts
them, imperfectly. New projects should start in HCL2.

**HCP Packer** is HashiCorp's hosted registry that stores image *metadata* (not the
images themselves) — versions, channels, ancestry — so Terraform can resolve "the
latest production AMI" by channel rather than hardcoded ID. It is an optional
commercial layer on top of the open-source CLI.

## Production Notes

**Licensing is the first thing to check.** In August 2023 HashiCorp relicensed
Packer (with the rest of its portfolio) from MPL-2.0 to the Business Source License
1.1[^3]. Building your own images is unaffected; the non-compete clause matters only
if you embed Packer in a competing commercial product. GitHub's license detector
reports this as `NOASSERTION`, but the repository `LICENSE` is BUSL-1.1.

**"Identical" requires discipline.** Pin the `source_ami_filter`/base image, pin
plugin versions (an unbounded `~>` can pull a breaking plugin), and pin package
versions in provisioners. Otherwise builds drift silently between runs.

**Orphaned cloud resources.** Each build spins up real, billable instances,
volumes, key pairs, and security groups. The default `-on-error=cleanup` tears them
down on failure, but a hard kill (SIGKILL, CI runner eviction) leaves orphans. Long
term, budget for a periodic sweep of `packer-*` tagged resources.

**Secrets leak into logs.** Mark variables `sensitive = true`; even so, verbose
`PACKER_LOG=1` output and some provisioner echoes can expose values. Never run debug
logging in CI with real credentials in scope.

**Slow feedback loop.** Every iteration provisions a full machine, so a failing
shell command 20 minutes in is expensive. Use the `breakpoint` provisioner and
`packer build -debug` to pause and inspect, and prototype provisioning logic against
the `docker` builder before running the slow cloud builder.

**Windows/WinRM builds are fragile** — bootstrap scripts, `winrm` firewall opening,
and Sysprep sequencing are common failure points and platform-specific.

## When to Use / When Not

**Use when:**
- You want immutable, pre-baked golden images instead of boot-time provisioning.
- You need the *same* image across multiple clouds/hypervisors from one template.
- You bake AMIs in CI to cut instance boot time and remove provisioning from the hot
  path.
- You already run Terraform/Vagrant and want a native image-production step.

**Avoid when:**
- You are container-first: a `Dockerfile` with BuildKit is more idiomatic than
  Packer's `docker` builder for most container workflows.
- You manage long-lived, mutable fleets and need ongoing configuration convergence —
  that is Ansible/Chef/Puppet's job, not image baking.
- You rebuild rarely enough that a hand-taken snapshot is cheaper than maintaining
  templates and plugins.

## Alternatives

- ansible/ansible — configuration management for running machines; use it alone when
  you converge mutable fleets rather than bake immutable images (and note Packer
  often calls Ansible as a provisioner).
- moby/moby — for container images, a Dockerfile is the more idiomatic path than
  Packer's docker builder.
- kubernetes-sigs/image-builder — actually a Packer + Ansible wrapper; use it when
  you specifically need Kubernetes node images.
- NixOS/nixpkgs — declarative, reproducible system images without imperative
  provisioners; use when bit-for-bit reproducibility is the priority.
- AWS EC2 Image Builder — a managed, single-cloud pipeline; use when you are
  AWS-only and prefer a hosted service over a self-run CLI.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2013-06 | Initial release by Mitchell Hashimoto[^1]. |
| 1.0.0 | 2017-10 | First stable release. |
| 1.5.0 | 2019-12 | HCL2 template support introduced alongside JSON. |
| 1.7.0 | 2021-03 | External plugin ecosystem; `packer init` and `required_plugins`. |
| 1.8.0 | 2022-05 | Bundled plugins removed; JSON templates deprecated in favor of HCL2. |
| 1.9.2 | 2023-08 | First release under the BUSL-1.1 license[^3]. |

## References

[^1]: Mitchell Hashimoto, "Packer" announcement — 2013-06-24. https://www.hashicorp.com/blog/packer
[^2]: Packer documentation, "Packer Terminology / Components". https://developer.hashicorp.com/packer/docs/terminology
[^3]: HashiCorp, "HashiCorp adopts Business Source License" — 2023-08-10. https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license

## Tags

go, infrastructure-as-code, immutable-infrastructure, machine-images, devops, cloud, hashicorp, hcl, ami, image-builder, cli
