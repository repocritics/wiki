# ahmetb/kubectx

> Two small CLIs — `kubectx` and `kubens` — that switch the current kubectl context and namespace faster than editing kubeconfig by hand.

[GitHub repo](https://github.com/ahmetb/kubectx) ·
[Official website](https://kubectx.dev) ·
[License: Apache-2.0](https://github.com/ahmetb/kubectx/blob/master/LICENSE)

## Overview

`kubectx` and `kubens` are companion command-line tools that wrap the two most
repetitive actions in day-to-day `kubectl` use: picking which cluster/context is
active, and picking which namespace that context defaults to. Both are thin
conveniences over the kubeconfig file — anything they do can be done with
`kubectl config use-context` and `kubectl config set-context --current
--namespace`, but the shorthand, tab-completion, and interactive fuzzy-picker
remove enough friction that the tools have become close to standard in the
Kubernetes operator's toolbox. As of 2026 the repo sits near 19.9k stars with
active maintenance (last push mid-2026)[^1].

The project began as two POSIX shell scripts and was later rewritten in Go[^2],
which is the implementation shipped today. The single most important thing to
understand about both tools is that they mutate your kubeconfig file in place:
switching context or namespace is a global side effect, not a per-shell one.
That design choice is the source of both their simplicity and their most common
footgun (covered in Production Notes). The author, Ahmet Alp Balkan, maintains
the project independently of the upstream Kubernetes project, though the tools
are also distributed as official `kubectl krew` plugins (`ctx` and `ns`)[^3].

## Getting Started

```bash
# macOS / Linux
brew install kubectx
# as kubectl plugins via krew
kubectl krew install ctx ns
```

```sh
# switch to a context by name
$ kubectx minikube
Switched to context "minikube".

# switch back to the previous context
$ kubectx -

# rename a verbose GKE context to something typeable
$ kubectx dublin=gke_ahmetb_europe-west1-b_dublin

# change the active namespace of the current context
$ kubens kube-system

# go back to the previous namespace
$ kubens -
```

With [`fzf`](https://github.com/junegunn/fzf) on your `PATH`, running `kubectx`
or `kubens` with no argument opens an interactive fuzzy picker instead of
printing the list. Both tools ship bash/zsh/fish completion.

## Architecture / How It Works

Each tool is a small Go binary that loads the kubeconfig (respecting
`$KUBECONFIG`, including colon-separated multi-file lists, falling back to
`~/.kube/config`), parses the YAML, edits one field, and writes it back:

- **kubectx** rewrites the top-level `current-context` field. Naming a context
  switches to it; `-` swaps to the previously active one; `name=old` renames a
  context entry; `-d name` deletes it.
- **kubens** looks up the current context's cluster and sets the `namespace`
  field on that context, so subsequent `kubectl` commands default to it.

The "previous" value that powers `kubectx -` / `kubens -` is persisted to a
small state file outside kubeconfig, so the swap survives across separate
invocations and shells. Interactive mode is not built in — it is delegated
entirely to `fzf` when present, which keeps the binaries dependency-free.

Two later additions soften the global-mutation model by scoping a switch to a
single shell instead of the shared kubeconfig: `kubectx -s NAME` launches a
subshell with an isolated, temporary kubeconfig pinned to one context, and
`kubectx -r NAME` launches a read-only shell where write verbs are blocked.
These exist precisely because the default behavior is global.

## Production Notes

**The switch is global, not per-terminal.** Because both tools edit the shared
kubeconfig, running `kubectx prod` in one terminal changes the active context
for *every* shell that reads that file. This is the number-one operational
hazard: an operator switches to `prod` in a background tab, forgets, and later
runs a destructive `kubectl delete` in a different tab believing they are on
staging. Mitigate with `kubectx -s`/`-r` isolated shells, a context/namespace
indicator in your prompt (kube-ps1, oh-my-posh, starship), or a per-shell
`$KUBECONFIG` discipline. Tools like `kubie` were built specifically to avoid
this class of mistake by isolating context to the shell that set it.

**YAML round-tripping is lossy.** The tools parse and re-emit kubeconfig, which
can reorder keys and strip comments and custom formatting from a hand-curated
`~/.kube/config`. If you keep annotated kubeconfigs under version control, expect
noisy diffs after a switch.

**No fuzzy search without fzf.** Interactive mode silently degrades to a plain
list if `fzf` is absent — not an error, just less useful. Set
`KUBECTX_IGNORE_FZF=1` to force list behavior when `fzf` is installed but
unwanted.

**Naming and packaging drift.** Under krew the commands are `kubectl ctx` and
`kubectl ns`, not `kubectx`/`kubens`; scripts and docs that assume the standalone
binary names will not match a krew-only install. Distro packages (apt, etc.) have
historically lagged the upstream release, so `--version` behavior and newer flags
like `-s`/`-r` may be missing on older packaged builds — prefer Homebrew, krew,
or a Releases-page binary when you need current features.

## When to Use / When Not

**Use when:**
- You juggle multiple clusters or namespaces and want fewer keystrokes than
  `kubectl config` subcommands.
- You want tab-completion or `fzf`-driven selection of long GKE/EKS context
  names.
- You want a zero-config, dependency-light addition that installs from every
  major package manager or as a kubectl plugin.

**Avoid / reconsider when:**
- You run many terminals against different clusters at once — the global switch
  is a real safety hazard; reach for a shell-isolating tool instead.
- You manage dozens of separate kubeconfig files and want to browse across them;
  a dedicated kubeconfig manager fits better.
- You want a full cluster UI (logs, exec, resource editing) — a TUI covers
  context switching plus much more.

## Alternatives

- sbstp/kubie — spawns a subshell per context/namespace so a switch is scoped to
  that shell, not global; use it when the shared-kubeconfig footgun bites.
- derailed/k9s — full-screen terminal UI for Kubernetes; use it when you want
  context/namespace switching bundled with live resource browsing.
- danielfoehrKn/kubeswitch — drop-in kubectx replacement built for hundreds of
  kubeconfig files across many stores; use it at large multi-cluster scale.
- sunny0826/kubecm — merge, split, and manage multiple kubeconfig files; use it
  when config-file wrangling, not just switching, is the pain.
- Plain `kubectl config use-context` / `set-context --current --namespace` — use
  when you can't add tooling and just need the built-in equivalent.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-03 | First public release as POSIX shell scripts (`kubectx`, `kubens`)[^1]. |
| Go rewrite | 2020 | Reimplemented in Go for speed, cross-platform builds, and completion[^2]. |
| krew plugins | — | Distributed as `kubectl ctx` / `kubectl ns` via krew[^3]. |
| isolated/read-only shells | — | `-s` (single-context subshell) and `-r` (read-only shell) added to contain the global-switch hazard. |

## References

[^1]: GitHub API — ahmetb/kubectx repository metadata (created 2017-03-30, ~19.9k stars, 1.38k forks, Apache-2.0, last push 2026-06-28). https://github.com/ahmetb/kubectx
[^2]: kubectx README — "This repository provides both `kubectx` and `kubens` tools"; Go implementation CI and installation. https://github.com/ahmetb/kubectx/blob/master/README.md
[^3]: Krew plugin index — `kubectl krew install ctx` / `ns`. https://github.com/kubernetes-sigs/krew

## Tags

go, kubernetes, kubectl, cli, kubectl-plugin, devops, context-switching, namespaces, developer-tools, krew
