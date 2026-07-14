# mikefarah/yq

> A single static Go binary that applies jq-style expressions to YAML — and, increasingly, to JSON, XML, TOML, CSV, and a dozen other formats.

[GitHub repo](https://github.com/mikefarah/yq) ·
[Official docs](https://mikefarah.gitbook.io/yq/) ·
[License: MIT](https://github.com/mikefarah/yq/blob/master/LICENSE)

## Overview

`yq` is a command-line processor for YAML and related structured formats, written in Go and distributed as a dependency-free static binary[^1]. It borrows jq's expression language — path traversal (`.a.b[0].c`), pipes, `select`, `map`, `reduce` — but reimplements it against a YAML document model rather than calling jq. The draw is operational: one binary you can drop into a container, a CI runner, or a Kubernetes init step to read and mutate config without a scripting runtime.

The defining tension is that `yq` looks like jq but is not jq. It implements a large, useful subset of jq's operators plus YAML-specific ones (comments, anchors, styles, tags), but the two evaluators diverge on edge cases, and expressions are not always portable between them[^2]. The second, sharper tension is a naming collision: `yq` also refers to kislyuk/yq, a Python tool that shells out to real jq. The two are unrelated, take incompatible syntax, and are frequently installed by the wrong package name — a recurring source of CI breakage.

As of this writing the project has roughly 15.7k stars and is actively maintained, with commits landing within the last week and a steady release cadence on the v4 line[^1]. It is one of the most widely vendored CLI tools in DevOps pipelines, shipped via Homebrew, apt/apk, Nix, Docker Hub, and a GitHub Action.

## Getting Started

```bash
brew install yq                          # macOS / Linux
# or a single binary, no package manager:
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 \
  -O /usr/local/bin/yq && chmod +x /usr/local/bin/yq
```

```bash
# Read a nested value
yq '.metadata.name' deployment.yaml

# Update in place, preserving comments and formatting
yq -i '.spec.replicas = 3' deployment.yaml

# Inject an environment variable (avoids quoting/interpolation bugs)
REPLICAS=5 yq -i '.spec.replicas = env(REPLICAS)' deployment.yaml

# Convert JSON to pretty YAML
yq -p json -o yaml < data.json

# Create a document from nothing (-n = null input)
yq -n '.name = "svc" | .port = 8080'
```

Note the v4 argument order: the expression comes first, then the file (`yq '.a' f.yaml`). This is the reverse of v3 and of many people's muscle memory.

## Architecture / How It Works

`yq` parses input into the `yaml.Node` AST from the go-yaml v3 library[^3], runs the expression against that tree, and re-serializes it. Keeping the node tree — rather than decoding to plain Go maps — is what lets `yq` preserve comments, key order, anchors, and scalar styles across an edit. It also means `yq`'s YAML fidelity is bounded by go-yaml's: quirks in how that library round-trips comments and whitespace surface directly in `yq` output.

The expression engine is a hand-written evaluator over an operator set (traverse, assign, `select`, `map`, `reduce`, `with`, arithmetic, string, boolean, path, etc.). Each operator is a node in an expression tree that transforms a stream of context nodes. Non-YAML formats are handled by pluggable encoder/decoder pairs: JSON, XML, TOML, CSV/TSV, properties, HCL, INI, base64, Lua, and a "kyaml" mode all flow through the same evaluator, so `.a.b = "x"` works identically regardless of the wire format. Format is auto-detected by extension and overridable with `-p`/`-o`.

Two evaluation modes matter. The default `eval` (`yq '...'`) applies the expression to each document of each file in sequence. `eval-all` (`yq ea '...'`) loads every document of every file into memory first, exposing them as one stream — required for cross-file merges and reductions like `. as $item ireduce ({}; . * $item)`.

The practical consequence of the go-yaml coupling and the bespoke evaluator is that `yq` is excellent at surgical, formatting-preserving edits of hand-authored config, and merely adequate as a general jq replacement. Anything relying on precise jq semantics should be tested, not assumed.

## Production Notes

- **The two-yq problem.** `apt install yq` on some distros, and `pip install yq`, give you kislyuk/yq (a jq wrapper with different syntax). Scripts written for mikefarah's tool then fail cryptically. Pin the install source; the Go tool is `yq-go` in Nix/Alpine and `go-yq` in Arch. Check `yq --version` prints `mikefarah`.
- **v3 → v4 is a hard break.** v3 used subcommands (`yq r file.yaml a.b.c`, `yq w -i ...`); v4 replaced them with jq-style expressions. There is no compatibility flag. Long-lived scripts, Ansible snippets, and blog copy-paste from before ~2020 will not run on current `yq`[^4].
- **YAML 1.1 vs 1.2 booleans.** `yq` assumes YAML 1.2, where `yes`/`no`/`on`/`off` are plain strings, not booleans[^5]. Files authored for 1.1 parsers (older Ruby/Symfony/Norway-problem configs) can change meaning on round-trip. Quote booleans you care about.
- **Comment and whitespace preservation is best-effort.** It handles common cases well but is explicitly documented as imperfect; blank-line placement and some head/foot comments can shift on write. Review diffs before committing machine-edited config.
- **Number and string coercion.** Large integers, leading-zero values, and version-like strings (`1.10`, `07`) are areas where YAML typing surprises people. Use `strenv`/`env` and explicit `| tostring` when you need a value kept verbatim.
- **snap strict confinement.** The snap package cannot read files outside `$HOME`; editing `/etc/...` requires piping through `sudo cat ... | yq ... | sudo sponge`.
- **Container image runs as non-root.** Since PR #860 the Docker image runs as user `yq`; mounts and in-image `apk add` need `--user=root` or a `USER root` layer. The alpine base also omits timezone data, so the `tz` operator needs `apk add tzdata`.
- **Security flags for untrusted expressions.** `--security-disable-file-ops` and `--security-disable-env-ops` disable `load()` and `env()`/`strenv()` respectively — relevant when an expression string comes from outside your trust boundary.

## When to Use / When Not

**Use when:**
- You need to read or mutate YAML/JSON config in shell, CI, Dockerfiles, or Kubernetes manifests without a Python/Node runtime.
- You want formatting- and comment-preserving in-place edits (`-i`) of hand-maintained files.
- You need lightweight format conversion (YAML ↔ JSON ↔ XML ↔ TOML ↔ CSV) with one tool.

**Avoid when:**
- You need full, spec-exact jq semantics — use jq on JSON directly rather than relying on `yq`'s subset.
- You're doing heavy programmatic transformation inside an app — a native YAML library in your language is more maintainable than shelling out.
- Bit-exact YAML round-tripping is a hard requirement; go-yaml's comment/whitespace handling is not lossless.
- Your team can't guarantee which `yq` is installed on target machines.

## Alternatives

- jqlang/jq — the original; use it when your data is JSON and you want exact, well-specified semantics.
- kislyuk/yq — Python wrapper that pipes YAML/XML/TOML through real jq; use it when you specifically want jq's engine with a YAML front-end (and accept the Python dependency).
- kubernetes-sigs/kustomize — use instead when the goal is structured, declarative overlays of Kubernetes manifests rather than ad-hoc edits.
- mikefarah/yq via `yq ea` vs. helm/helm templating — use Helm when you need packaged, parameterized release charts, not one-off value tweaks.
- itchyny/gojq — pure-Go jq reimplementation; use when you want jq semantics embeddable in a Go program or a single binary and your data is JSON.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2015 | Initial release; subcommand style (`yq r/w/d`). |
| 3.x | 2019–2020 | Last of the subcommand-based line; widely referenced in older docs. |
| 4.0 | 2020 | Full rewrite to jq-style expression syntax; hard break from v3[^4]. |
| 4.x | 2021–2026 | Added XML, TOML, CSV/TSV, properties, HCL, INI, Lua, kyaml formats; security flags; non-root container image. Active. |

## References

[^1]: mikefarah/yq repository and README. https://github.com/mikefarah/yq
[^2]: yq documentation — operator reference and jq comparison notes. https://mikefarah.gitbook.io/yq/
[^3]: go-yaml v3, the underlying YAML parser providing the `yaml.Node` AST. https://github.com/go-yaml/yaml/tree/v3
[^4]: yq v3-to-v4 upgrade guide. https://mikefarah.gitbook.io/yq/upgrading-from-v3
[^5]: YAML 1.2 specification (plain `yes`/`no` are strings, not booleans). https://yaml.org/spec/1.2.2/

## Tags

go, cli, yaml, json, xml, jq, devops-tools, config-processing, static-binary, format-conversion
