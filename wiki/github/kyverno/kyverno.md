# kyverno/kyverno

> A Kubernetes-native policy engine that expresses admission-control policy as Kubernetes resources instead of a separate policy language.

[GitHub repo](https://github.com/kyverno/kyverno) ·
[Official website](https://kyverno.io) ·
[License: Apache-2.0](https://github.com/kyverno/kyverno/blob/main/LICENSE)

## Overview

Kyverno is a policy engine for Kubernetes that validates, mutates, generates, and cleans up cluster resources, and verifies container image signatures for supply-chain integrity[^1]. It was created at Nirmata in 2019, entered the CNCF Sandbox in 2020, and is a CNCF Incubating project[^2]. Its target audience is platform-engineering and security teams that want guardrails (Pod Security Standards, image provenance, naming/label conventions, default network policies) enforced at admission time and audited on a schedule.

The defining design choice is "no new language required." Policies are ordinary Kubernetes custom resources (`ClusterPolicy`, `Policy`) written in YAML, using pattern matching, overlays, JMESPath expressions, and — in recent versions — CEL. This is the central tradeoff versus its main competitor, OPA Gatekeeper, which uses Rego: Kyverno is far more approachable for teams already fluent in Kubernetes YAML, but complex conditional logic expressed as YAML plus JMESPath becomes verbose and hard to read, where a real expression language would be terser. Kyverno also carries more runtime machinery (several in-cluster controllers, dynamically registered webhooks, generated report objects) than a minimal validating webhook.

Kyverno complements rather than replaces Kubernetes RBAC and the native `ValidatingAdmissionPolicy` / `MutatingAdmissionPolicy` primitives — it can even generate native `ValidatingAdmissionPolicy` objects from Kyverno policies — while adding reporting, exceptions, background scanning, and generate/mutate-existing capabilities the built-ins lack[^1].

## Getting Started

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

A minimal policy that blocks Pods running as root:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-run-as-nonroot
spec:
  validationFailureAction: Enforce   # Enforce = block; Audit = report only
  background: true                    # also scan existing resources
  rules:
    - name: check-runasnonroot
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Containers must set securityContext.runAsNonRoot: true"
        pattern:
          spec:
            containers:
              - securityContext:
                  runAsNonRoot: true
```

Apply with `kubectl apply -f policy.yaml`. Test policies offline against manifests with the `kyverno` CLI (`kyverno apply policy.yaml --resource pod.yaml`) — this is the recommended way to validate policy behavior in CI before it reaches a live webhook.

## Architecture / How It Works

Kyverno runs inside the cluster as a set of controllers and registers admission webhooks dynamically based on the policies you install — a policy that matches only Pods produces a webhook scoped to Pods, so the API server is not called for resource types no policy cares about.

Since the 1.10 rearchitecture, the single controller was split into separate, independently scalable deployments[^3]:

- **Admission controller** — serves the validating/mutating webhooks on the request hot path.
- **Background controller** — handles `generate` and `mutate-existing` rules and reconciles the resources they own.
- **Reports controller** — produces `PolicyReport` / `ClusterPolicyReport` objects from background scans.
- **Cleanup controller** — runs `CleanupPolicy` TTL/cron deletions.

Rule types map to distinct behaviors: `validate` accepts or rejects (or audits) a request; `mutate` patches the incoming object via strategic-merge or JSON patch overlays; `generate` creates and keeps in sync downstream resources (e.g. a default `NetworkPolicy` per new namespace); `verifyImages` checks cosign/notary signatures and attestations. Matching uses `match`/`exclude` blocks with `any`/`all` conditions, and rules can reference external data via API calls, ConfigMaps, or the image registry. CEL-based rules and native `ValidatingAdmissionPolicy` generation were added to align with upstream Kubernetes policy primitives.

The tight coupling to admission webhooks is the source of both its power and its operational risk: everything hinges on the webhook being reachable, correct, and fast.

## Production Notes

**Webhook failure mode is the biggest footgun.** With `failurePolicy: Fail`, if the Kyverno admission controller is unavailable, the API server rejects the matched admission requests — which can block deployments cluster-wide, including Kyverno's own recovery. Exclude system namespaces (`kube-system`, the Kyverno namespace) and think carefully about `Fail` vs `Ignore` per policy. A Kyverno outage that also blocks its own Pods from scheduling is a classic self-inflicted cluster lockout.

**Resource footprint.** Background scanning, policy reports, and admission review caching can consume significant memory on large or high-churn clusters; OOMKills of the reports/background controllers have been a recurring operational theme. Right-size requests/limits, and be deliberate about which policies set `background: true`.

**Report volume.** One `PolicyReport` object is produced per namespace (aggregating results), and violations across thousands of workloads generate a lot of etcd-backed objects. Policy Reporter (a companion project) or metrics/alerting is usually needed to make the output actionable rather than just voluminous.

**Upgrade pain.** Kyverno moves fast and has shipped breaking changes across minor versions. The 1.10 controller split changed the deployment topology and Helm values; policy CRD fields have been renamed and relocated across releases (validation-action placement, CLI flags, CEL adoption). Read the release notes and migration guides for every minor bump — do not assume a values.yaml or a policy manifest carries forward unchanged. Pin the Helm chart version, and test policies with the CLI against the target Kyverno version before upgrading the cluster.

**Latency.** Every matched create/update pays a network round-trip to the webhook. Keep policies narrowly scoped, avoid unnecessary external API/registry lookups on the hot path, and watch webhook timeout settings — a slow `verifyImages` call against a rate-limited registry can stall admissions.

## When to Use / When Not

**Use when:**
- Your team writes Kubernetes YAML and does not want to learn Rego or a separate policy DSL.
- You need mutate, generate, and background-scan behaviors, not just validation.
- You want image signature/attestation verification (cosign, notary) enforced at admission.
- You want turnkey Pod Security Standards, CIS, or supply-chain policy libraries.

**Avoid when:**
- Your policy decisions span many systems beyond Kubernetes (APIs, CI, Envoy) — a general policy engine fits better.
- You only need simple field validation and want zero extra controllers — native `ValidatingAdmissionPolicy` (CEL) is built into modern Kubernetes.
- You cannot tolerate the operational surface of in-cluster webhooks and their failure modes.
- Your logic is deeply conditional/procedural; it will read more cleanly in Rego or WASM policies than in YAML + JMESPath.

## Alternatives

- open-policy-agent/gatekeeper — the main competitor; use when your org standardizes on OPA/Rego and wants constraint templates.
- open-policy-agent/opa — general-purpose policy engine; use when policy decisions extend well beyond Kubernetes admission.
- kubewarden/kubewarden-controller — WASM-based policies written in Rust/Go/etc; use when you want policies compiled from real languages.
- aquasecurity/trivy — scanning of images, IaC, and misconfigurations; use for shift-left CI checks rather than live cluster enforcement.
- falcosecurity/falco — runtime threat detection; complementary, use for runtime rather than admission-time control.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019 | Created at Nirmata[^2]. |
| — | 2020-11 | Accepted into the CNCF Sandbox[^2]. |
| — | 2022-07 | Promoted to CNCF Incubating[^2]. |
| 1.10 | 2023 | Rearchitecture: split into admission / background / reports / cleanup controllers[^3]. |
| 1.11 | 2023 | CEL support and native `ValidatingAdmissionPolicy` generation added. |
| 1.12–1.14 | 2024–2025 | Ongoing CEL expansion, reporting and CLI improvements (exact release dates unverified). |

## References

[^1]: Kyverno README and documentation — capabilities, non-goals, and companion projects. https://kyverno.io/docs/
[^2]: CNCF project profile, Kyverno. https://www.cncf.io/projects/kyverno/
[^3]: Kyverno docs, "High Availability" / controller architecture (1.10 split into separate controllers). https://kyverno.io/docs/high-availability/

## Tags

kubernetes, policy-as-code, admission-controller, go, security, compliance, governance, cncf, cloud-native, webhook, supply-chain
