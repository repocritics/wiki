# opentofu/opentofu

> An MPL-licensed, community-governed fork of Terraform, created in response to HashiCorp's 2023 relicensing.

[GitHub repo](https://github.com/opentofu/opentofu) ·
[Official website](https://opentofu.org) ·
[License: MPL-2.0](https://github.com/opentofu/opentofu/blob/main/LICENSE)

## Overview

OpenTofu is a declarative infrastructure-as-code tool: you describe cloud
resources in HCL, it computes an execution plan against real provider APIs, and
applies the diff. It exists because HashiCorp changed Terraform's license from
MPL-2.0 to the Business Source License (BSL 1.1) on 2023-08-10[^1]. Within weeks
a group of vendors and users forked the last MPL-licensed Terraform under the
"OpenTF" manifesto; the project was renamed OpenTofu and moved under the Linux
Foundation in September 2023[^2]. The first general-availability release, 1.6.0,
shipped in January 2024[^3].

The defining tension is compatibility versus divergence. OpenTofu began as a
near drop-in replacement — same HCL, same `.tf` files, state-file compatible with
Terraform 1.5/1.6, most commands identical down to the flags. But the two
codebases have diverged since the fork: OpenTofu ships features Terraform lacks
(client-side state encryption, early variable evaluation), and Terraform
continues under BSL with its own additions. The longer both live, the less "just
swap the binary" holds, and migration in the Terraform-to-OpenTofu direction is
far better supported than the reverse. It is aimed at teams that want Terraform's
model without the BSL license: vendors building products on top of an IaC engine,
regulated shops needing permissive licensing, and anyone wary of a single vendor
controlling the tool their infrastructure depends on.

## Getting Started

```bash
# macOS (Homebrew)
brew install opentofu
# or the standalone installer
curl --proto '=https' --tlsv1.2 -fsSL https://get.opentofu.org/install-opentofu.sh | sh
```

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"   # resolved via registry.opentofu.org
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "example" {
  bucket = "opentofu-demo-bucket-1234"
}
```

```bash
tofu init      # download providers, configure backend
tofu plan      # show the execution plan
tofu apply     # apply after confirmation
```

The CLI is `tofu`, not `terraform`. The `terraform {}` configuration block keeps
its name for compatibility.

## Architecture / How It Works

OpenTofu is a Go binary built around three stages that are unchanged from
Terraform's design:

1. **Configuration → graph.** HCL is parsed into resources, data sources,
   modules, variables, and outputs, then compiled into a directed acyclic graph
   keyed on interpolation dependencies. Non-dependent nodes are walked in
   parallel.
2. **Plan.** OpenTofu reconciles desired config against prior **state** (a JSON
   document recording real resource IDs and attributes) and provider-refreshed
   reality, producing a create/update/delete/replace plan.
3. **Apply.** The plan is executed against **providers** — separate plugin
   binaries that speak a gRPC protocol over `go-plugin`. Providers own all API
   calls; OpenTofu core never talks to a cloud directly.

Providers and modules resolve through **registry.opentofu.org**, a decentralized
registry that maps `namespace/name` to GitHub releases rather than hosting
artifacts centrally[^4]. This is a deliberate divergence from Terraform's
`registry.terraform.io`. For the common providers (AWS, Google, Azure, Kubernetes)
the same upstream code is served, but the two registries are distinct namespaces.

Features unique to OpenTofu since the fork:

- **State encryption** (1.7) — client-side encryption of state and plan files at
  rest, with pluggable key providers (PBKDF2, AWS/GCP KMS, OpenBao/Vault). Plain
  Terraform stores state unencrypted; this is the flagship differentiator[^5].
- **Early variable / backend evaluation** (1.8) — variables and locals usable in
  `backend`, `module source`, and `provider` blocks, which Terraform historically
  disallowed.
- **`-exclude`** (1.9) — the inverse of `-target`, to apply everything except
  named resources.

State backends (S3, GCS, azurerm, Postgres, HTTP, Kubernetes, local) are the same
family Terraform ships.

## Production Notes

**State compatibility is one-way in practice.** OpenTofu can adopt a Terraform
state file, but once you use OpenTofu-only features (state encryption, early
evaluation, provider iteration), the state and config no longer round-trip back
to Terraform cleanly. Treat the migration as a decision, not an experiment, and
pin your tool version in CI.

**Migrating from Terraform > 1.6 is not guaranteed.** The clean path is Terraform
1.5/1.6 → OpenTofu 1.6. Configurations that use Terraform 1.7+ features (some
`import` block behavior, provider-defined functions ergonomics, newer built-ins)
may need adjustment. Do not assume a modern Terraform project drops in unchanged.

**Registry namespace differences bite in air-gapped and enterprise setups.**
Modules or providers referenced by explicit `registry.terraform.io/...` source
addresses do not automatically resolve against the OpenTofu registry. Private
providers published only to HCP/HashiCorp will not appear in OpenTofu's registry;
you either mirror them, use a network mirror, or a filesystem mirror.

**State locking is still your responsibility.** As with Terraform, concurrent
`apply` against an unlocked backend corrupts state. Use a backend that supports
locking (S3 + DynamoDB historically, or S3 native lockfile on newer setups) and
never point two pipelines at the same unlocked state.

**Provider licensing is separate from OpenTofu's.** OpenTofu core is MPL-2.0, but
individual providers carry their own licenses. Some HashiCorp-published providers
have themselves moved to BSL; using OpenTofu does not relicense the providers you
depend on.

**No first-party managed backend.** Terraform has HCP Terraform / Terraform Cloud
for remote state, runs, and policy. OpenTofu is CLI-only; remote execution, run
pipelines, and drift detection come from third parties (Spacelift, env0, Scalr,
Terrateam) or hand-rolled CI — the most common operational gap after switching.

## When to Use / When Not

**Use when:**
- You need Terraform's model but require a permissive (MPL-2.0) license.
- You are a vendor or platform building a product on top of an IaC engine and
  cannot ship BSL-covered code.
- You want state encryption at rest without bolting on external tooling.
- You want community/foundation governance rather than single-vendor control.

**Avoid when:**
- You depend on HCP Terraform / Terraform Cloud, Sentinel policy, or HashiCorp's
  commercial support — those are Terraform-only.
- Your project already uses Terraform 1.7+ features and you have no reason to
  move; the migration cost may exceed the license benefit.
- You want infrastructure defined in a general-purpose language rather than HCL —
  Pulumi or the CDKs fit better.
- Your team is small and standardized on the `terraform` toolchain with no
  licensing constraint; the switch buys little.

## Alternatives

- hashicorp/terraform — the upstream. Use it when you need HCP Terraform,
  Sentinel, or a commercial support contract, and BSL is acceptable.
- pulumi/pulumi — use when you want IaC in TypeScript/Python/Go instead of HCL,
  with real loops and abstractions.
- getcrossplane/crossplane — use when your control plane is Kubernetes and you
  want reconciliation loops rather than imperative `apply` runs.
- gruntwork-io/terragrunt — not a replacement but a wrapper; use to keep large
  multi-environment OpenTofu/Terraform setups DRY.
- ansible/ansible — use for procedural provisioning and configuration management
  rather than declarative resource-graph reconciliation.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2023-08-10 | HashiCorp relicenses Terraform to BSL 1.1[^1]. |
| — | 2023-08-25 | OpenTF manifesto and fork announced[^2]. |
| — | 2023-09 | Renamed OpenTofu; joins the Linux Foundation[^2]. |
| 1.6.0 | 2024-01 | First GA release; drop-in fork of Terraform 1.5/1.6[^3]. |
| 1.7.0 | 2024-05 | Client-side state encryption; provider-defined functions[^5]. |
| 1.8.0 | 2024-09 | Early variable / backend / provider evaluation. |
| 1.9.0 | 2024-12 | `-exclude` flag; expanded variable validation. |

## References

[^1]: HashiCorp, "HashiCorp adopts Business Source License" — 2023-08-10. https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license
[^2]: Linux Foundation, "Announcing OpenTofu" — 2023-09-20. https://www.linuxfoundation.org/press/announcing-opentofu
[^3]: OpenTofu blog, "OpenTofu 1.6.0 is now available." https://opentofu.org/blog/opentofu-1-6-0/
[^4]: OpenTofu Registry. https://github.com/opentofu/registry
[^5]: OpenTofu docs, "State encryption." https://opentofu.org/docs/language/state/encryption/

## Tags

infrastructure-as-code, terraform, opentofu, devops, go, hcl, cloud, provisioning, mpl-2.0, fork, state-management
