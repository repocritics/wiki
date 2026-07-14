# hashicorp/vagrant

> Declarative, reproducible development VMs from a single Ruby file — the tool that made "works on my machine" a shared machine.

[GitHub repo](https://github.com/hashicorp/vagrant) ·
[Official website](https://www.vagrantup.com) ·
[License: BUSL-1.1](https://github.com/hashicorp/vagrant/blob/main/LICENSE)

## Overview

Vagrant builds and configures portable development environments — historically full
virtual machines — from a declarative `Vagrantfile`. Written by Mitchell Hashimoto and
first released in 2010, it predates the company HashiCorp itself and became the project
the company was founded around[^1]. The pitch has stayed constant for over a decade:
check a `Vagrantfile` into your repo, run `vagrant up`, and every developer (and CI)
gets a byte-for-byte identical Linux box regardless of host OS.

Vagrant is a thin, opinionated orchestration layer over other people's virtualization.
It does not ship a hypervisor. Instead it drives *providers* — VirtualBox by default,
plus VMware, Hyper-V, Docker, libvirt, and Parallels via plugins — and hands the
booted machine to *provisioners* (shell, Ansible, Chef, Puppet, Salt) to install
software. Its defining tension is that it inherited the mid-2010s "a VM per project"
model just as the industry moved to containers and, later, cloud dev environments.
Vagrant still solves a real problem — full-kernel, multi-machine, non-Linux-guest
environments that containers cannot represent — but it is no longer the default answer
for "give me a clean Ubuntu to hack in."

Since HashiCorp's 2023 license change the code is under the Business Source License
(BUSL-1.1) rather than an OSI-approved license[^2], and HashiCorp is now an IBM
subsidiary following the acquisition that closed in 2025[^3]. The project is maintained
but conservative: releases are infrequent and mostly compatibility and bug-fix driven.

## Getting Started

Install VirtualBox (free, cross-platform), then install Vagrant from the official
packages. `bsdtar` and `curl` must be on your `PATH`.

```bash
vagrant init hashicorp/bionic64   # writes a Vagrantfile
vagrant up                         # downloads box, boots the VM
vagrant ssh                        # shell into it
vagrant halt                       # graceful shutdown
vagrant destroy                    # delete the VM
```

```ruby
# Vagrantfile — Ruby DSL, evaluated top to bottom
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.synced_folder ".", "/vagrant", type: "rsync"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus   = 2
  end

  config.vm.provision "shell", inline: <<-SHELL
    apt-get update && apt-get install -y nginx
  SHELL
end
```

## Architecture / How It Works

The `Vagrantfile` is not config data — it is a Ruby program evaluated at load time. This
is powerful (loops, conditionals, ENV reads, multi-machine definitions) and a footgun
(a `Vagrantfile` can do anything Ruby can, and errors surface as Ruby stack traces).

Three plugin interfaces do the real work:

- **Providers** translate the abstract machine into a concrete VM. VirtualBox is
  bundled behavior; VMware, Hyper-V, libvirt, Docker, and Parallels are separate
  plugins. Provider parity is uneven — features and box formats that work on
  VirtualBox often need per-provider handling elsewhere.
- **Provisioners** run after boot to converge the box to a desired state (shell scripts,
  or full config-management via Ansible/Chef/Puppet/Salt).
- **Synced folders** mount host directories into the guest. The default VirtualBox
  shared-folder driver is notoriously slow for large trees; NFS, rsync, and SMB are the
  standard escapes.

Environments are reproduced from **boxes** — pre-packaged base images distributed
through the HashiCorp/Vagrant Cloud registry (`owner/name` slugs like `hashicorp/bionic64`).
A box pins the guest OS; the `Vagrantfile` pins everything layered on top.

A long-running effort to re-architect Vagrant's core in Go — a client/daemon model with
a gRPC plugin protocol, replacing the Ruby engine — lived on the main branch for years
but never shipped as a stable successor; current releases remain the Ruby
implementation[^4]. Treat any "Vagrant is going Go" expectation as unfulfilled.

## Production Notes

**Apple Silicon is the dominant pain point.** VirtualBox — the free default and the
target of nearly every published box — has no first-class arm64 support, and the vast
majority of boxes on the registry are amd64-only. On M-series Macs you are pushed toward
VMware Fusion, Parallels, or QEMU-based providers, and toward the small set of arm64
boxes. Teams with mixed Intel/Apple-Silicon laptops cannot assume one `Vagrantfile`
boots identically for everyone. This single issue drives much of the migration to Lima,
containers, and Docker Desktop.

**VirtualBox version coupling.** Vagrant and VirtualBox are separately versioned and
regularly desynchronize: a new VirtualBox release frequently breaks `vagrant up` until
Vagrant ships a patch (or vice versa). Pin both, and read release notes before upgrading
either.

**Synced-folder performance.** Default VirtualBox shared folders make large codebases
(node_modules, vendored deps) painfully slow inside the guest. Switch to `type: "nfs"`
or `type: "rsync"`; NFS needs host-side privileges and can require sudoers entries.

**Plugin fragility.** Plugins (vagrant-libvirt, vagrant-vmware-desktop, vagrant-disksize,
vagrant-hostmanager, etc.) install into Vagrant's embedded Ruby and break across Vagrant
upgrades. Plugin version conflicts are a recurring support burden; there is no lockfile
equivalent for the plugin set.

**Licensing.** BUSL-1.1 permits production use but forbids offering Vagrant as a hosted
or embedded service that competes with HashiCorp/IBM[^2]. For most internal dev-env use
this is irrelevant; for anyone building a product *around* Vagrant it is a real
constraint versus the pre-2023 MIT terms.

**Box hygiene.** Boxes are snapshots and drift from upstream security patches. A team
that pins an old box for reproducibility is also pinning old, unpatched packages;
`vagrant box update` and periodic re-provisioning are on you.

## When to Use / When Not

**Use when:**
- You need a full VM with its own kernel — kernel modules, systemd, non-Linux guests,
  or OS behavior a container cannot fake.
- You orchestrate multi-machine topologies (app + db + load balancer) on one laptop.
- You want a single checked-in file that boots identically for a homogeneous
  (typically x86_64) team and matches a config-management workflow.

**Avoid when:**
- Your team is on Apple Silicon and your boxes are amd64 — you will fight emulation.
- A container is sufficient: Docker/Podman/devcontainers are lighter, faster, and the
  ecosystem default for language-runtime dev environments.
- You want cloud-hosted or ephemeral environments — Codespaces, Gitpod, and remote
  devcontainers cover that space directly.

## Alternatives

- hashicorp/packer — sibling tool for *building* box/VM images; complementary to
  Vagrant rather than a replacement, but often what you actually need if the goal is
  image production.
- canonical/multipass — use instead when you just want a quick Ubuntu VM, especially on
  macOS/Windows, without a Vagrantfile.
- lima-vm/lima — use instead on Apple Silicon Macs where you need Linux VMs with good
  arm64 and file-sharing behavior.
- devcontainers/spec — use instead when your environment fits in a container and you
  want VS Code / Codespaces integration.
- containers/podman — use instead for rootless, daemonless containerized dev
  environments where a full VM is overkill.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2010-03 | First public release by Mitchell Hashimoto[^1]. |
| 1.0 | 2012-03 | First stable release; project that HashiCorp was founded around. |
| 1.1 | 2013-03 | Provider plugin architecture; VirtualBox becomes one provider among many, VMware support added. |
| 2.0 | 2017-10 | Major version; stability and plugin-system consolidation. |
| 2.2 | 2018-09 | Long-lived 2.2.x line; triggers, improved Windows/Hyper-V support. |
| 2.3 | 2022-09 | Continued maintenance; Go re-architecture work in progress on main[^4]. |
| 2.4 | 2023-11 | Releases under BUSL-1.1 following HashiCorp's license change[^2]. |
| 2.4.9 | 2025-08 | Recent maintenance release[^5]. |

## References

[^1]: Vagrant history and origin — HashiCorp. https://www.hashicorp.com/products/vagrant
[^2]: HashiCorp, "HashiCorp adopts Business Source License" (2023-08-10); Vagrant LICENSE file (BUSL-1.1). https://www.hashicorp.com/license-faq
[^3]: IBM completes acquisition of HashiCorp (closed 2025). https://newsroom.ibm.com/2025-02-27-ibm-completes-acquisition-of-hashicorp
[^4]: Vagrant Go re-architecture / `vagrant serve` daemon work, main branch. https://github.com/hashicorp/vagrant
[^5]: Vagrant releases. https://github.com/hashicorp/vagrant/releases

## Tags

ruby, virtualization, developer-tools, virtual-machines, devops, vagrant, hashicorp, provisioning, dev-environments, infrastructure-as-code
