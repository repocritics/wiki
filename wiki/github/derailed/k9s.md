# derailed/k9s

> A terminal UI for navigating, observing, and managing live Kubernetes clusters.

[GitHub repo](https://github.com/derailed/k9s) ·
[Official website](https://k9scli.io) ·
[License: Apache-2.0](https://github.com/derailed/k9s/blob/master/LICENSE)

## Overview

k9s is a full-screen terminal application that sits on top of a Kubernetes
cluster and renders its resources as live, keyboard-driven tables. It was
started in 2019 by Fernand Galiana (the `derailed` handle) and remains a
largely single-maintainer project funded through GitHub Sponsors[^1]. Where
`kubectl` is a request/response CLI, k9s is a continuously-watching TUI:
resource lists refresh in place, and every view exposes context-specific
actions (describe, edit, logs, shell, port-forward, delete) behind single-key
bindings.

The intended user is an operator or developer who spends the day inside a
cluster and wants to move faster than typing `kubectl get`/`describe`/`logs`
loops. k9s is aliased navigation: `:pod`, `:svc`, `:deploy`, `:ctx` jump
between resource types, `/` filters, and drill-downs (a deployment to its
pods to their containers to their logs) happen without leaving the keyboard.

The defining tension is that k9s is a *convenience layer over the same
client-go machinery `kubectl` uses*, not a separate control plane. It inherits
your kubeconfig, your RBAC, and your cluster's API behavior verbatim. That
makes it safe and predictable, but it also means k9s is only ever as fast,
as permitted, and as reliable as the underlying API server and your
credentials allow — and its always-on watch model puts more sustained load on
both the client and the cluster than intermittent `kubectl` calls do.

## Getting Started

```shell
# macOS / Linux
brew install derailed/k9s/k9s

# Windows
winget install k9s

# Go toolchain (builds HEAD, not a stable release)
go install github.com/derailed/k9s@latest
```

Launch against your current kubeconfig context, or scope it up front:

```shell
k9s                          # current context / default namespace
k9s -n my-namespace          # start in a namespace
k9s --context staging        # pick a context
k9s --readonly               # disable all mutating commands
```

Inside the UI, everything is a command or a key:

```text
:pod⏎          list pods in the active namespace
:pod /web⏎     list pods filtered by name "web"
:ctx⏎          switch clusters/contexts
l              view logs for the selected resource
s              shell into a container (pods)
d              describe   e  edit   ctrl-d  delete
?              show context-sensitive key bindings
```

## Architecture / How It Works

k9s is written in Go and built on **tcell** and **tview** — terminal
cell/widget libraries that Galiana maintains his own forks of, so the
rendering stack co-evolves with the app rather than tracking upstream[^2].

Cluster data comes from **client-go informers**. When you open a resource
view, k9s establishes a watch and maintains an in-memory cache that the API
server keeps up to date via the watch stream; the table you see is a
projection of that cache, re-rendered on change. This is why lists update
without polling — and why memory and watch count scale with the breadth of
resources you browse, not with how often you refresh.

Notable subsystems:

- **Aliases / commands** — resource types are addressed by singular, plural,
  short name, or custom alias (`po`, `pods`, `pod`). CRDs are discovered from
  the API server and are navigable the same way as built-in kinds.
- **XRay** — a tree view (`:xray RESOURCE`) that renders ownership graphs,
  e.g. deployment to ReplicaSet to pods, with status roll-ups.
- **Pulses** — a dashboard (`:pulses`) of cluster-wide health metrics.
- **Popeye** — an in-app cluster sanitizer view (`:popeye`), reusing
  Galiana's separate Popeye project to flag misconfigurations[^3].
- **Benchmarking** — a built-in HTTP load generator (via the `hey` library)
  for port-forwarded services.
- **Extensibility files** — plugins, hotkeys, aliases, skins, and custom
  column views are all user-editable YAML under the k9s config dirs. `k9s
  info` prints their resolved paths.

Configuration lives under XDG paths (`~/.config/k9s`, `~/.local/share/k9s`,
`~/.local/state/k9s` on Linux; the exact roots print from `k9s info`). Skins
recolor the entire UI; the tool targets a 256-color terminal.

## Production Notes

**Memory scales with what you watch.** Each resource view spins up informers
that cache the full watched set. On large clusters, browsing high-cardinality
resources across all namespaces (all pods, all events) can push k9s memory
into hundreds of MB and add real watch load to the API server. Scoping to a
namespace materially reduces both.

**RBAC gaps surface as errors, not graceful degradation.** k9s shows exactly
what your credentials permit. Kinds you cannot `list`/`watch` render as
access errors; using it with a narrow ServiceAccount is fine but expect empty
or errored views for forbidden resources. Nothing k9s does bypasses RBAC —
mutations go through the same authorization as `kubectl`.

**Config churn across 0.x releases.** k9s has never shipped a 1.0; it moves
fast and the config schema has been reorganized more than once — most notably
the context/config restructuring around the v0.31 line[^4]. Skins and config
files written for an older version can silently stop applying or need
migration after an upgrade. Pin the version in shared tooling and re-check
your skin/config after bumps.

**Kubernetes version skew.** k9s bundles a client-go version and prefers
recent Kubernetes servers (the project recommends 1.28+). The README carries a
compatibility matrix mapping k9s releases to client versions; running a very
old k9s against a new cluster (or vice-versa) can produce missing kinds or
decode errors.

**It is a foreground TUI, not automation.** k9s is for interactive operation.
`--readonly` (also enforceable via config/env for locked-down environments) is
the right default for production access; there is no headless/scriptable mode
— reach for `kubectl` in CI. Destructive keys are fast: `ctrl-d` deletes with
a confirm, `ctrl-k` kills with no dialog. On a real cluster the muscle memory
matters.

## When to Use / When Not

**Use when:**
- You live in a cluster and want fast interactive navigation, logs, and shells.
- You're triaging incidents and need to pivot across resources quickly.
- You want a read-only, RBAC-respecting observation console for on-call.
- You already use `kubectl` and want the same semantics with less typing.

**Avoid when:**
- You need automation or scripting — k9s has no non-interactive mode.
- You're on a very large cluster and can't afford the watch/memory footprint
  of broad, all-namespace views.
- You want a graphical dashboard with charts and multi-cluster panes — a web
  or desktop UI fits better.
- You need vendor-supported tooling; k9s is a community, largely
  single-maintainer project.

## Alternatives

- kubernetes/kubectl — the canonical CLI; use when you need scripting,
  automation, or a stable non-interactive interface.
- kdash-rs/kdash — Rust TUI dashboard with a similar concept; use when you
  prefer a Rust binary, accepting a smaller feature set and community.
- lensapp/lens — desktop GUI IDE for clusters; use when you want graphical
  dashboards and multi-cluster views over a terminal.
- stern/stern — multi-pod log tailing only; use when log aggregation is all
  you need, alongside `kubectl`.
- derailed/popeye — cluster sanitizer/linter (also integrated into k9s); use
  standalone for scheduled config audits rather than live navigation.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2019-01 | First public release; tcell/tview TUI over client-go[^1]. |
| v0.30.0 | 2023-11 | Filtered/labeled/context query syntax for resource views[^5]. |
| v0.31.0 | 2024-01 | Config/context restructuring[^4]. |
| v0.40.0 | 2025 | "Column Blow" release[^5]. |

(k9s tags frequently within the 0.x series; consult the releases page for the
exact current version and its client-go compatibility.)

## References

[^1]: derailed/k9s README and GitHub Sponsors note. https://github.com/derailed/k9s
[^2]: derailed maintains forks of tcell and tview used by k9s. https://github.com/derailed/tview
[^3]: Popeye — Kubernetes cluster sanitizer, integrated as a k9s view. https://github.com/derailed/popeye
[^4]: k9s configuration and skins documentation. https://k9scli.io
[^5]: k9s releases (demo/sneak-peek videos referenced in README). https://github.com/derailed/k9s/releases

## Tags

go, kubernetes, kubernetes-cli, terminal-ui, tui, cluster-management, devops, k8s, client-go, observability, sre
