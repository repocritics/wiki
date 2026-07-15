# stern/stern

> Multi-pod, multi-container log tailing for Kubernetes, with per-source color coding and live pod discovery.

[GitHub repo](https://github.com/stern/stern) ·
[License: Apache-2.0](https://github.com/stern/stern/blob/master/LICENSE)

## Overview

Stern is a command-line tool that tails logs from many Kubernetes pods and containers at once. You give it a query — either a regular expression matched against pod names, or a `<resource>/<name>` reference such as `deployment/nginx` — and it opens a log stream for every matching container, colorizing each pod/container pair so interleaved output stays readable[^1]. Pods that appear after startup are picked up automatically; pods that are deleted drop out of the tail. This "follow the workload, not a single pod" behavior is the core difference from `kubectl logs`.

The project is a community fork of `wercker/stern`, the original tool written at Wercker (later acquired by Oracle), which stopped being maintained. The fork was created in October 2020[^2] and has since become the de facto continuation: it migrated to modern `client-go`, added JSON and Go-template output, a config file, resource-based queries, and interactive selection. It is packaged as a `kubectl` plugin via Krew (`kubectl krew install stern`) as well as Homebrew, asdf, and WinGet.

The defining tradeoff is scope: stern is a *viewer*, not a log pipeline. It reads live logs directly from the Kubernetes API and prints them to your terminal. It stores nothing, indexes nothing, and survives no restart of the pod whose logs it was showing. That narrowness is exactly why it is fast to reach for during an incident — and why it is the wrong tool for retention, search-after-the-fact, or alerting.

## Getting Started

```bash
# Homebrew
brew install stern
# or as a kubectl plugin via Krew
kubectl krew install stern
# or from source
go install github.com/stern/stern@latest
```

```bash
# Tail every container in every pod of the nginx deployment
stern deployment/nginx

# Tail pods whose name matches a regex, across all namespaces
stern "web-\w" --all-namespaces

# Tail one container, since 15 minutes ago, with timestamps
stern auth -c gateway --since 15m -t

# Pipe structured logs to jq
stern backend -o json | jq .
```

The first positional argument is a regex when it is not of the form `<resource>/<name>`; `stern nginx` matches any pod containing "nginx", while `stern pod/nginx-abc123` resolves an exact pod. Supported resource kinds include `pod`, `deployment`, `replicaset`, `statefulset`, `daemonset`, `job`, `service`, and `replicationcontroller`.

## Architecture / How It Works

Stern is a thin, concurrency-heavy client over the Kubernetes logs API. The pipeline:

1. **Target resolution.** The query is turned into a label/field selector or resolved through a resource reference. Stern uses `client-go` to `list` and `watch` pods matching the target, so the set of tailed containers is maintained live rather than snapshotted once.
2. **Stream fan-out.** For each matching container, stern opens the pod `GetLogs` streaming endpoint (`follow=true` by default). Every stream runs in its own goroutine. These are ordinary Kubernetes API log requests — the same ones `kubectl logs -f` makes — served by the API server proxying to the node kubelet.
3. **Multiplexing and color.** Output from all streams is merged onto stdout. Each pod and container is assigned a color from a configurable SGR palette so lines from different sources are visually separable. Because streams are independent, lines are not globally time-ordered unless you add `--timestamps` and sort downstream.
4. **Formatting.** Each line is rendered through a Go `text/template`. Predefined templates (`default`, `raw`, `json`, `extjson`, `ppextjson`) cover common cases; `--template`/`--template-file` expose a set of helper functions (`parseJSON`, `toRFC3339Nano`, `levelColor`, color helpers) for reshaping structured logs inline.

Two knobs govern load and semantics. `--max-log-requests` caps concurrent streams to protect the cluster; its default and behavior change with `--no-follow` (50 with an error on overflow when following, 5 as a hard limit when not). `--no-follow` flips stern from a live tail into a bounded dump that exits once existing logs are shown — useful with `--only-log-lines` and `sort` to produce time-ordered output across pods.

## Production Notes

**It hits the API server and kubelets, once per container.** A broad query with `--all-namespaces` can open hundreds of concurrent log streams. Each is a real request flowing API server → kubelet. On large clusters this is a genuine load source; keep `--max-log-requests` sane and prefer narrow selectors. This is the most common way stern surprises operators.

**Regex-vs-resource is a footgun.** `stern api` is a substring regex and will also match `api-gateway`, `legacy-api`, and anything else containing "api". Use `stern deployment/api` or anchored patterns (`stern '^api-'`) when precision matters. During an incident the loose default match can quietly pull in unrelated workloads.

**No history, no persistence.** Stern shows what the kubelet still has on disk for the *current* container instance. Rotated logs are gone, and logs from a previous crashed container are not shown (there is no equivalent of `kubectl logs --previous`). If you need post-mortem or retained logs, you need a real aggregation layer, not stern.

**Ordering is per-stream, not global.** Because each container is a separate goroutine and stream, interleaved lines are in arrival order, not event order. For a time-sorted view use `--since=5m --no-follow --only-log-lines -A -t . | sort -k4`; live-follow mode cannot give you a globally sorted stream.

**RBAC.** Stern needs `get`/`list`/`watch` on `pods` and `get` on `pods/log` in every targeted namespace. `--all-namespaces` against a cluster where your account lacks cluster-wide read will fail per-namespace rather than up front.

**Defaults worth knowing.** `--since` defaults to 48h, so a fresh tail may replay two days of backlog before catching up — set `--tail 0` to start from now. Color is `auto` (off when stdout is not a TTY); force it with `--color always` when piping into a pager that understands ANSI.

## When to Use / When Not

**Use when:**
- You are debugging a live workload and want every replica's logs in one colorized stream.
- You want a `kubectl logs` that follows a deployment/daemonset instead of a single pod name.
- You need quick inline reshaping of JSON logs (`-o json | jq`, or `--template` with `parseJSON`).
- You want a single static binary or `kubectl` plugin with no server-side component.

**Avoid when:**
- You need retained, searchable logs after the fact — use an aggregation stack.
- You need globally time-ordered logs across many pods in real time.
- You must inspect logs of an already-crashed previous container instance.
- You are on a very large cluster and a wide tail would add meaningful API-server load.

## Alternatives

- grafana/loki — persistent log aggregation with LogQL; use when you need retention, search, and alerting rather than an ephemeral live tail.
- boz/kail — similar Go-based multi-pod tailer with label/deployment matchers; use when you prefer its selector model.
- johanhaleby/kubetail — older Bash wrapper around `kubectl logs`; use only where you cannot install a binary and want something dependency-light.
- wercker/stern — the original, now unmaintained; use nothing here, it exists only as history.
- kubernetes/kubectl (`kubectl logs`) — built in, no extra install; use when you only need one pod/container and no live discovery.

## History

| Version | Date | Notes |
|---------|------|-------|
| wercker/stern | 2016 | Original tool at Wercker; multi-pod tailing with color. |
| — | 2017 | Wercker acquired by Oracle; upstream maintenance stalls. |
| stern/stern fork | 2020-10 | Community fork begins; migrates to modern `client-go`[^2]. |
| 1.x | 2020–2021 | Resource queries (`<resource>/<name>`), config file, JSON/template output. |
| 1.x | 2022–2024 | `--no-follow`, `--stdin`, `--timezone`, interactive `--prompt`, extended template functions. |

Exact per-release dates are not pinned here; consult the [releases page](https://github.com/stern/stern/releases) for the authoritative changelog.

## References

[^1]: stern README — usage, query semantics, and output templates. https://github.com/stern/stern#readme
[^2]: stern/stern repository, created 2020-10-21 as a fork of the discontinued wercker/stern. https://github.com/stern/stern

## Tags

kubernetes, logging, cli, go, devops, observability, debugging, log-tailing, kubectl-plugin, containers
