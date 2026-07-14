# spf13/viper

> Go configuration library that merges defaults, files, environment variables, flags, and remote key/value stores into one keyspace.

[GitHub repo](https://github.com/spf13/viper) ·
[pkg.go.dev](https://pkg.go.dev/github.com/spf13/viper) ·
[License: MIT](https://github.com/spf13/viper/blob/master/LICENSE)

## Overview

Viper is a Go configuration library written by Steve Francia (spf13), the same author as Cobra; the two are designed as companions (Cobra for CLI structure, Viper for the configuration behind it)[^1]. It sits underneath a large share of the Go CLI/infra ecosystem — Hugo, Docker Notary, DigitalOcean's doctl, Coder, Meshery, and Vitess all read config through it. Its selling point is one API over many sources: it reads JSON, TOML, YAML, INI, envfile, and Java Properties files, plus environment variables, pflag flags, `io.Reader` buffers, and remote stores (etcd, Consul, Firestore, NATS), then flattens them into a single case-insensitive keyspace with a fixed precedence order.

The defining tension is that this convenience comes from an opinionated, lossy merge model. Every key is lowercased and every source is collapsed into one flat map, so Viper is easy to bolt onto a project early and awkward to remove or constrain later. It does not deep-merge nested values, it cannot distinguish keys that differ only by case, and `Unmarshal` decodes via `mapstructure` tags rather than the `json`/`yaml` tags most structs already carry. These are not bugs so much as consequences of the "registry for all your config" design, and they are why the project has spent recent years prioritizing stability and a still-forming v2 over new features[^2].

Viper is mature and widely depended-on rather than actively evolving. The README carries a standing v2 feedback request, and several long-standing behaviors (the global singleton, case-insensitivity) are candidates for change only in that future major version[^3].

## Getting Started

```shell
go get github.com/spf13/viper
```

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
)

func main() {
	viper.SetConfigName("config")      // config.yaml/.json/.toml — extension inferred
	viper.AddConfigPath("/etc/appname/")
	viper.AddConfigPath("$HOME/.appname")
	viper.AddConfigPath(".")

	viper.SetDefault("port", 8080)
	viper.SetEnvPrefix("app")           // env vars read as APP_*
	viper.AutomaticEnv()

	if err := viper.ReadInConfig(); err != nil {
		var notFound viper.ConfigFileNotFoundError
		if !errors.As(err, &notFound) {
			panic(fmt.Errorf("fatal error config file: %w", err))
		}
		// no config file found is acceptable; defaults + env still apply
	}

	fmt.Println(viper.GetInt("port"))
}
```

## Architecture / How It Works

Viper is a `*Viper` struct holding one map per source (defaults, config, overrides, env bindings, pflag bindings, aliases, key/value cache) plus a merged view. Reads walk those maps in a fixed precedence: explicit `Set` > flags > environment > config file > remote KV > defaults. Every lookup is resolved live, not cached — env vars and bound flags are read at access time, so `viper.GetString` reflects the current process environment on each call.

Keys are normalized to lowercase and nested access uses `.` as the delimiter (`GetString("datastore.metric.host")`), with numeric segments indexing into slices. A literal flat key that matches the whole dotted path wins over the nested structure, which is a subtle source of surprise. The delimiter is configurable via `NewWithOptions(viper.KeyDelimiter("::"))` when real keys contain dots.

File parsing is pluggable through an encoding registry: each format (YAML, TOML, JSON, etc.) is a codec, and `SetConfigType` forces a type when reading from a stream that has no extension. Unmarshaling into structs goes through `github.com/go-viper/mapstructure` — a maintained fork of `mitchellh/mapstructure` that Viper adopted — using `mapstructure:"..."` struct tags and decode hooks (the same mechanism used to parse custom string formats)[^4]. Live reload is built on `fsnotify`: `WatchConfig` starts a goroutine that fires the `OnConfigChange` callback on write events. Remote support is a separate blank-imported package (`viper/remote`) so the core has no hard dependency on etcd/Consul client libraries; encrypted remote values are decrypted through `crypt` against a GPG keyring.

## Production Notes

**Case-insensitivity is the headline footgun.** Every key is lowercased internally, so a config that relies on two keys differing only by case will silently collapse to one, and round-tripping (`AllSettings` → marshal → write) loses the original casing. Environment variables are case-*sensitive* at the OS level but map onto lowercased Viper keys, which is why `SetEnvKeyReplacer` exists to bridge `SCREAMING_SNAKE_CASE` env names to kebab/dotted keys. Teams integrating with case-sensitive backends (Kubernetes-style config, some remote stores) hit this repeatedly.

**No deep merge.** Overriding a nested value replaces the entire branch, not just the changed leaf — the README states this explicitly. `MergeConfigMap` and multi-file merging are shallow at the level they operate on, so layering a partial override file over a base file does not behave like a recursive merge.

**Unmarshal uses `mapstructure` tags, not `json`/`yaml`.** Structs shared with an HTTP API or a YAML marshaller need a second set of `mapstructure:"..."` tags, or the field names must match after lowercasing. `,squash` handles embedded structs; forgetting it silently drops embedded fields.

**The global singleton is discouraged and may be deprecated.** Viper ships a package-level instance for convenience, but the maintainers recommend constructing `viper.New()` and passing it explicitly, because the singleton makes tests order-dependent and state leaky. Deprecation is tracked in issue #1855[^3].

**Config watching is `fsnotify`-shaped.** Editors that save via atomic rename (write temp + rename over the target) can emit remove/rename events instead of write, causing the watcher to fire twice or stop watching the inode. Watch behavior across container bind-mounts and network filesystems is inconsistent for the same reason.

**Upgrade friction.** The move to `go-viper/mapstructure/v2` and the encoding-registry refactor changed some decode and format-handling internals; pin versions and test unmarshaling when upgrading across those. Recent releases require a current Go toolchain (the README targets Go >= 1.23), so Viper can pull your module's minimum Go version forward.

## When to Use / When Not

**Use when:**
- You are building a Cobra CLI and want flags, env, and config files unified with one precedence model.
- You need to accept many config formats and sources without writing the plumbing.
- You want live config reload or remote KV config with minimal setup.
- Config is broadly flat and case is not semantically meaningful.

**Avoid when:**
- Key case matters, or you need deep/recursive merging of layered config.
- You want a single typed struct as the source of truth with standard `json`/`yaml` tags — a thinner loader (koanf, or plain `encoding/*` + `env`) fits better.
- You want a small dependency footprint; Viper pulls in parsers, `fsnotify`, and (optionally) remote clients.
- You need strict, explicit configuration with no hidden global state.

## Alternatives

- knadh/koanf — same "merge many sources" idea, smaller modular core, case-preserving, no forced global; the most direct like-for-like replacement.
- kelseyhightower/envconfig — env-vars-only into a struct via tags; use when config is 12-factor env and nothing else.
- caarlos0/env — minimal struct-tag env parsing; use when you want types without a config framework.
- ilyakaznacheev/cleanenv — env + file into one struct with defaults; use for small services wanting a single typed config.
- go standard library `flag` + `encoding/json`/`yaml` — use when the config surface is small enough that a framework is overhead.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-04 | First commit; released as a Cobra companion by spf13[^1]. |
| v1.0.0 | 2018-06 | First tagged stable release. |
| v1.7.0 | 2020-05 | Encoding registry / codec refactor era; broader format handling. |
| v1.8.0 | 2021-03 | Finder and config-type handling improvements. |
| v1.18.0 | 2023-12 | Adopted `go-viper/mapstructure` fork; slog-based logging. |
| v1.19.0 | 2024-07 | Maintenance release; dependency and format updates. |
| v1.20.0 | 2025 | Recent maintenance line; Go >= 1.23 baseline. |

## References

[^1]: Viper README, "Why is it called Viper?" — designed as a companion to Cobra. https://github.com/spf13/viper#faq
[^2]: Viper README FAQ, "prioritizing backwards compatibility and stability over features." https://github.com/spf13/viper#i-found-a-bug-or-want-a-feature-should-i-file-an-issue-or-a-pr
[^3]: Viper issue #1855 — global instance deprecation discussion. https://github.com/spf13/viper/issues/1855
[^4]: go-viper/mapstructure — maintained fork used by Viper for struct decoding. https://github.com/go-viper/mapstructure

## Tags

go, configuration, config-management, environment-variables, cli, cobra, twelve-factor, yaml, toml, remote-config
