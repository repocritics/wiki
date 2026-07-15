# knadh/koanf

> A dependency-light configuration library for Go that treats every config source and every format as a pluggable module.

[GitHub repo](https://github.com/knadh/koanf) ·
[License: MIT](https://github.com/knadh/koanf/blob/master/LICENSE)

## Overview

koanf reads configuration from heterogeneous sources — files, environment
variables, command-line flags, S3, Vault, Consul, etcd, raw bytes, Go maps and
structs — and merges them into a single queryable instance. It was written by
Kailash Nadh (CTO of Zerodha) and has been public since 2019[^1]. Its explicit
pitch is to be a lighter alternative to spf13/viper: the same job, but with
cleaner interface boundaries and, critically, without dragging viper's large
transitive dependency tree into every build.

The whole design rests on two interfaces. A `Provider` yields configuration
(either raw bytes for a parser, or an already-nested `map[string]any`); a
`Parser` turns raw bytes into that nested map. Everything else — twelve-odd
source integrations and nine-odd format parsers — is an implementation of one
of those two interfaces, and each lives in its own Go module under
`providers/` and `parsers/`. You install only the ones you use.

That modularity is the defining tension. viper gives you one import and
everything works; koanf gives you a small core and asks you to `go get` each
provider and parser separately, sometimes with `/v2` version suffixes that do
not match the core's version. The payoff is a dependency graph you can actually
account for; the cost is more import lines and a real chance of version-suffix
confusion during setup.

## Getting Started

```shell
# Core (note the /v2 module path).
go get -u github.com/knadh/koanf/v2

# Each provider and parser is a separate module.
go get -u github.com/knadh/koanf/providers/file
go get -u github.com/knadh/koanf/parsers/yaml
```

```go
package main

import (
	"fmt"
	"log"

	"github.com/knadh/koanf/v2"
	"github.com/knadh/koanf/parsers/json"
	"github.com/knadh/koanf/parsers/yaml"
	"github.com/knadh/koanf/providers/file"
)

// "." is the key-path delimiter; it can be any string.
var k = koanf.New(".")

func main() {
	if err := k.Load(file.Provider("mock.json"), json.Parser()); err != nil {
		log.Fatalf("load: %v", err)
	}
	// Merge YAML on top of the JSON already loaded.
	k.Load(file.Provider("mock.yml"), yaml.Parser())

	fmt.Println(k.String("parent1.name"))
	fmt.Println(k.Int("parent1.id"))
}
```

## Architecture / How It Works

Loading is always `k.Load(provider, parser)`. If the provider returns a nested
map directly (`confmap`, `structs`, `posflag`), the parser argument is `nil`;
if it returns bytes (`file`, `s3`, `rawbytes`), you pass a matching parser.
Internally koanf flattens everything to a single delimited keyspace, so
`parent1.child1.name` is one string key, and typed accessors (`String`, `Int`,
`Bool`, `Duration`, `Strings`, `Time`, …) read from that flat map. Keys are
case-sensitive: `app.port` and `APP.port` are distinct.

Merging is order-dependent and last-write-wins. Each successive `Load` deep-
merges nested maps and overwrites scalars/slices. There is no imposed source
precedence — you decide it by call order (typically defaults → file → env →
flags). `StrictMerge` (via `koanf.NewWithConf`) turns type conflicts into
errors instead of silent overwrites, but note that cross-format merges can then
fail on type mismatches, e.g. JSON decodes integers as `float64` while YAML
decodes them as `int`.

`Unmarshal` scans the flat keyspace into a struct using `koanf`-tagged fields;
under the hood this is mapstructure, so its decode-hook and weak-typing
behaviour applies. `UnmarshalConf{FlatPaths: true}` lets a flat target struct
pull keys from arbitrary nesting depths.

Some providers (`file`, `appconfig`, `vault`, `consul`) expose `Watch()` for
live reload via a callback. The README is explicit that this is not goroutine-
safe: if other goroutines call `Get*()` while the watch callback runs a
`Load()`, you must guard it yourself with a mutex — the library does not.

## Production Notes

- **Version-suffix sprawl is the main setup footgun.** The core is `/v2`, and
  several providers/parsers carry their own `/v2` (e.g. `env/v2`,
  `consul/v2`, `etcd/v2`, `toml/v2`) that is independent of the core version.
  Mixing a v1-era provider import with the v2 core, or omitting a `/v2` suffix,
  produces confusing build errors. Check the README's installable list for the
  exact path of each module you pull.
- **`Watch()` concurrency.** Live reload plus concurrent reads is a data race
  by default. The common safe pattern is to build a fresh `koanf.New(...)` in
  the callback and atomically swap a pointer, rather than mutating the live
  instance in place.
- **StrictMerge across formats.** Enabling it to catch config drift can
  backfire on mixed JSON/YAML/TOML stacks because parsers disagree on numeric
  types. Keep a single format per merge chain, or leave strict mode off and
  validate after load.
- **Env var mapping is manual.** koanf does not auto-bind env vars to keys;
  `env` provider takes a prefix plus a transform function you write to map
  `MYVAR_PARENT_CHILD` to `parent.child`. This is more explicit than viper's
  automatic binding and more code, but there is no hidden precedence magic.
- **No built-in required/validation layer.** koanf loads and reads; it does not
  enforce that a key exists or matches a schema. Validation is your job (or
  `Unmarshal` into a struct and check).

## When to Use / When Not

**Use when:**
- You want tight control over your dependency tree and object only to the
  sources/formats you actually use.
- You need to merge several config sources with an explicit, order-defined
  precedence.
- You are already avoiding viper specifically for its dependency weight.

**Avoid when:**
- You want a single import that does everything with zero wiring — viper is
  less code to start.
- Your config is one env-var struct or one file; a focused env/struct library
  or the format's own parser is simpler than koanf's abstraction.
- You need schema validation, required-key enforcement, or typed config
  generation out of the box — koanf leaves those to you.

## Alternatives

- spf13/viper — the incumbent; use it when you want batteries-included config
  with automatic env binding and don't mind a heavy dependency tree.
- ilyakaznacheev/cleanenv — use it when config is primarily env vars plus one
  file mapped straight into a struct with minimal wiring.
- caarlos0/env — use it when you only need env vars decoded into a struct.
- sethvargo/go-envconfig — use it for env-only config with context-aware
  loading and mutators, no file/merge machinery.
- BurntSushi/toml or go-yaml/yaml — use the format parser directly when you
  read exactly one file in one format and never merge sources.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-06 | First public release; single-module viper alternative[^1]. |
| v2 | 2023 | Major restructure: providers and parsers split into independent modules so external deps detach from the core[^2]. |

## References

[^1]: knadh/koanf repository and README. https://github.com/knadh/koanf
[^2]: koanf v2 module layout — installable providers/parsers listed in the README. https://github.com/knadh/koanf#api

## Tags

go, golang, configuration, config-management, config-loader, environment-variables, yaml, toml, viper-alternative, library
