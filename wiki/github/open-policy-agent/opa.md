# open-policy-agent/opa

> A general-purpose policy engine that decouples authorization and policy decisions from application code, using the Rego query language.

[GitHub repo](https://github.com/open-policy-agent/opa) ·
[Official website](https://www.openpolicyagent.org) ·
[License: Apache-2.0](https://github.com/open-policy-agent/opa/blob/main/LICENSE)

## Overview

Open Policy Agent (OPA) is a policy engine written in Go, started at Styra in 2016 and donated to the Cloud Native Computing Foundation, where it reached graduated status in 2021[^1]. Its premise is that policy — "who can do what," "which configs are allowed," "what must be true before this deploys" — should be pulled out of application code and expressed declaratively in one place. A service asks OPA a question with some JSON input; OPA evaluates rules plus data and returns a JSON answer. The answer can be a boolean, a set, an object, anything expressible in Rego.

The defining feature and the defining friction is **Rego**, OPA's policy language. Rego is a declarative, Datalog-derived language over JSON, not an imperative or general-purpose one. It is genuinely good at "given this document, does it satisfy these constraints," but its evaluation model (implicit iteration, partial-set semantics, everything-is-a-value-or-undefined) is unlike anything most engineers have written, and the learning curve is the single most cited complaint. Teams frequently adopt OPA for the decoupling and then spend their first months fighting Rego intuition.

OPA is domain-agnostic by design: it does not know about Kubernetes, Terraform, or HTTP. That neutrality is why it shows up across Kubernetes admission control, microservice API authorization, Terraform plan checks, CI/CD gates, and data filtering — but it also means OPA ships no policy; you or an ecosystem project supplies it.

## Getting Started

```bash
# macOS (Homebrew) — or download a static binary from GitHub releases
brew install opa
# Docker
docker run -p 8181:8181 openpolicyagent/opa run --server
```

```rego
# example.rego
package authz
import rego.v1

default allow := false

# allow if the user is an admin
allow if input.user.role == "admin"

# or if they own the resource and the action is read
allow if {
    input.action == "read"
    input.user.id == input.resource.owner
}
```

```bash
# evaluate one query against the policy + an input document
opa eval -d example.rego -i input.json 'data.authz.allow'
# or run as a server and POST inputs to the REST API
opa run --server example.rego
curl localhost:8181/v1/data/authz/allow -d '{"input": {...}}'
```

## Architecture / How It Works

OPA is a single static Go binary with no external dependencies. It runs in two fundamentally different shapes, and choosing wrong is a common early mistake:

1. **As a service (sidecar/host-level daemon).** `opa run --server` exposes a REST API on `:8181`. Applications POST an `input` document to `/v1/data/<path>` and get a decision back. This is the Kubernetes-admission and microservice-authz deployment. Latency is a network hop, usually co-located (localhost sidecar) to keep it sub-millisecond.
2. **As a Go library.** The `github.com/open-policy-agent/opa/rego` package embeds the evaluator in-process, no network hop. This is the path for Go services that want policy decisions without operating a separate component.

Rego evaluation compiles policy into an intermediate representation and evaluates it against a **`data` document** (external context you push in) and the query `input`. Two mechanisms matter operationally:

- **Bundles.** OPA polls (or is pushed) gzipped tarballs of policy + data from an HTTP server or OCI registry. This is how policy is distributed and versioned in production — you do not `kubectl edit` policy into a running OPA; you publish a new bundle and every OPA instance pulls it[^2]. Bundle serving is what Styra DAS and competing control planes sell.
- **Partial evaluation.** OPA can specialize a policy against known data and emit a residual — used to push policy decisions down into SQL WHERE-clauses (data filtering) or to compile Rego to Wasm for edge execution[^3].

For very high-throughput or non-Go environments, OPA supports **compiling policies to WebAssembly**, and the sibling project OPA can target a small Wasm runtime. Separately, the CNCF **Gatekeeper** project wraps OPA specifically for Kubernetes admission with CRD-based constraints — many people who "use OPA on Kubernetes" are actually running Gatekeeper, not OPA directly.

## Production Notes

**Rego v1 is the big migration.** OPA 1.0 (December 2024) made the `rego.v1` syntax the default: `if` and `contains` keywords became mandatory for rules, and several legacy constructs were removed[^4]. Pre-1.0 policies without `import rego.v1` will not behave identically. `opa fmt` and `opa check` help, but a policy corpus written across years of OPA versions is a real migration project, not a flag flip.

**Latency and memory scale with data, not just policy.** OPA holds the entire `data` document in memory. Pushing a large external dataset (all users, all resources, all org structure) into OPA to make decisions "self-contained" can blow up memory and bundle-download time. The common footgun is treating OPA as a database; the intended pattern is to keep decision-relevant data small, or use partial evaluation / external data fetching (`http.send`) judiciously — `http.send` inside evaluation adds unpredictable latency and is often disallowed.

**Decision logging is a cost center.** OPA can stream every decision (input + result) to a remote endpoint for audit. At high QPS this is significant network and storage volume; sample or mask it. Inputs frequently contain sensitive data (tokens, PII), so decision logs are a compliance surface, not just an ops one.

**Cold-start and bundle-availability coupling.** An OPA that cannot reach its bundle server on startup has no policy. Configure it to fail closed or serve from a last-known-good bundle deliberately — the default behavior and your desired behavior may differ, and "OPA allowed everything because it started with no policy" is a documented class of incident.

**Testing is a strength worth using.** Rego has a built-in unit-test framework (`opa test`), coverage, and a profiler (`opa eval --profile`). Policies that are not tested drift silently; the tooling to prevent that ships in the box, unlike much of the ecosystem.

## When to Use / When Not

**Use when:**
- You have policy logic duplicated across multiple services or languages and want one enforcement point and one language.
- You need Kubernetes admission control, API authorization, or IaC/CI policy checks and want a CNCF-standard, non-proprietary engine.
- You want policy as versioned, testable, distributable artifacts (bundles) rather than code changes.
- Decisions are a function of the request plus a bounded amount of context you can supply.

**Avoid when:**
- Your authorization model is simple RBAC that a library or your framework already covers — OPA is operational overhead you may not need.
- Decisions require large, rapidly-changing datasets or relationship graphs — a purpose-built system (Zanzibar-style) fits better than pushing all data into OPA.
- Your team has no appetite for learning Rego and the policies are trivial; the cognitive cost outweighs the decoupling.
- You need policy decisions inside a language with no good in-process option and cannot afford a network hop per decision.

## Alternatives

- openfga/openfga — use instead when you need relationship-based (ReBAC / Google Zanzibar-style) authorization over large graphs rather than rule evaluation over supplied JSON.
- casbin/casbin — use instead when you want an embedded access-control library with familiar RBAC/ABAC models and no separate Rego language to learn.
- open-policy-agent/gatekeeper — use instead of raw OPA specifically for Kubernetes admission, where you want CRD-defined constraints and templates rather than hand-managing OPA.
- cerbos/cerbos — use instead when you prefer YAML-defined, decoupled authorization for application access control without Rego.
- kyverno/kyverno — use instead of OPA/Gatekeeper on Kubernetes when you want policies written as native Kubernetes YAML resources rather than a separate policy language.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016 | Initial open-source release out of Styra. |
| — | 2018-03 | Accepted into CNCF Sandbox[^1]. |
| — | 2021-01 | Graduated in the CNCF[^1]. |
| 0.34 | 2021 | WebAssembly compilation and partial-eval maturity for edge/data-filtering use[^3]. |
| 1.0.0 | 2024-12 | `rego.v1` becomes default; `if`/`contains` mandatory; legacy syntax removed[^4]. |

## References

[^1]: CNCF, "Cloud Native Computing Foundation Announces Open Policy Agent Graduation" — 2021-02-04. https://www.cncf.io/announcements/2021/02/04/cloud-native-computing-foundation-announces-open-policy-agent-graduation/
[^2]: OPA documentation, "Bundles." https://www.openpolicyagent.org/docs/management-bundles
[^3]: OPA documentation, "Policy Language / Partial Evaluation" and Wasm. https://www.openpolicyagent.org/docs/wasm
[^4]: OPA blog, "OPA 1.0 Is Now Available." https://www.openpolicyagent.org/docs/v1-upgrade

## Tags

go, policy-engine, authorization, rego, cncf, cloud-native, kubernetes, admission-control, declarative, policy-as-code
