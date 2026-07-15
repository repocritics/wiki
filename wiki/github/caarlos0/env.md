# caarlos0/env

> Parses environment variables into Go structs using struct tags, with no third-party dependencies.

[GitHub repo](https://github.com/caarlos0/env) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/github.com/caarlos0/env/v11) ·
[License: MIT](https://github.com/caarlos0/env/blob/main/LICENSE.md)

## Overview

`env` is a small Go library that maps environment variables onto the fields of a
struct. You annotate fields with `env:"NAME"` tags, call `env.Parse(&cfg)`, and
the library reads `os.Environ()`, converts each string to the field's declared
type, and assigns it. It is the work of Carlos Alexandro Becker (`caarlos0`),
better known as the author of GoReleaser, and shares that project's house style:
narrow scope, conventional-commit history, zero external dependencies (it uses
only `reflect` and other standard-library packages)[^1].

The library is deliberately one-dimensional. It does not read `.env` files, YAML,
flags, or remote config stores; it does not validate business rules; it does not
watch for changes. It does exactly one job — string environment → typed struct —
and the maintainer has declared it **feature-complete**, with only bug fixes
planned going forward[^2]. That self-imposed ceiling is the project's defining
tradeoff: a stable, predictable dependency that will not churn, at the cost of
composing it with other tools (a dotenv loader, a validator) for anything beyond
parsing. As of 2026 it sits around 6.3k stars and is a common building block in Go
service configuration, including Encore's stack[^3].

## Getting Started

```bash
go get github.com/caarlos0/env/v11
```

The import path carries the major version (`/v11`) — a Go modules requirement,
and the main thing to get right when upgrading.

```go
package main

import (
	"fmt"
	"time"

	"github.com/caarlos0/env/v11"
)

type Config struct {
	Host     string        `env:"HOST" envDefault:"0.0.0.0"`
	Port     int           `env:"PORT" envDefault:"8080"`
	Hosts    []string      `env:"HOSTS" envSeparator:":"`
	Timeout  time.Duration `env:"TIMEOUT" envDefault:"5s"`
	Password string        `env:"PASSWORD,required,unset"`
}

func main() {
	cfg, err := env.ParseAs[Config]() // generics form; env.Parse(&cfg) also works
	if err != nil {
		panic(err)
	}
	fmt.Printf("%+v\n", cfg)
}
```

`ParseAs[T]()` (generics) and `Parse(&cfg)` are equivalent; use whichever reads
better. `env.Must(env.Parse(&cfg))` panics on error for main-package config.

## Architecture / How It Works

The whole library is reflection over struct fields. `Parse` walks the struct with
`reflect`, and for each exported field reads the `env` tag plus its siblings
(`envDefault`, `envPrefix`, `envSeparator`, `envKeyValSeparator`). It looks the
variable up in the environment map, then dispatches on the field's `reflect.Kind`
to a built-in parser. All Go built-in numeric/bool/string kinds are covered, plus
`time.Duration`, `time.Location`, `url.URL`, and any type implementing
`encoding.TextUnmarshaler`; pointers, slices, slices of pointers, and maps of the
above are handled recursively[^1].

Nested structs are parsed recursively, and `envPrefix` on a struct field prepends
a string to every variable name resolved inside it — the mechanism for grouping
(`DB_HOST`, `DB_PORT`) without repeating the prefix per field. For types the
library does not know, you register a parser function via the `FuncMap` option;
this is the extension point rather than an interface you implement.

The `env` tag takes comma-separated options that change resolution behavior rather
than type: `,required` (error if the variable is unset), `,notEmpty` (error if set
but empty — a distinct check), `,file` (treat the value as a path and read the
file's contents instead), `,expand` (interpolate `${OTHER_VAR}` references),
`,init` (allocate nil pointers), and `,unset` (delete the variable from the
process environment after reading, useful for secrets). `ParseWithOptions` exposes
knobs like a substitute `Environment` map (for tests), alternate tag names,
`RequiredIfNoDef`, `UseFieldNameByDefault`, and an `OnSet` hook.

Because everything is decided at run time by reflection, the struct tags are a
small untyped DSL the compiler cannot verify: a misspelled option or wrong
separator surfaces as a runtime error or silently wrong value, not a build failure.

## Production Notes

- **Unexported fields are silently ignored — by design and permanently.** A
  lowercase field with an `env` tag will never be populated, and you get no
  warning. This is the single most common surprise; the README flags it with a
  `CAUTION` admonition[^1].
- **Major-version import paths are upgrade friction.** Each breaking release bumps
  the module path (`/v10` → `/v11`), so upgrading is a find-and-replace across
  every import plus a `go.mod` change, not just `go get -u`. The upside is that a
  major bump can never silently change behavior on you.
- **`required` and `notEmpty` are different guarantees.** `,required` fails only
  when the variable is absent; an explicitly empty string satisfies it. Use
  `,notEmpty` when empty is also invalid. Mixing these up is a frequent config bug.
- **Defaults defeat `required`.** A field with `envDefault` is never "unset" from
  the parser's view, so `,required` on the same field is meaningless. For
  environment-specific requiredness, prefer the `RequiredIfNoDef` option.
- **No validation layer.** The library checks presence and type, nothing else.
  Range checks, enums, and cross-field rules need a separate validator
  (`go-playground/validator` is the usual pairing).
- **`,file` reads eagerly at parse time.** Handy for Docker/Kubernetes secret
  mounts, but the file must exist when `Parse` runs, and its full contents
  (including a trailing newline, if present) become the value.
- **Slice parsing is separator-splitting, not shell-quoting** — no escaping, so a
  value containing the separator can't be represented without changing it.

## When to Use / When Not

**Use when:**

- Your configuration is genuinely environment-variable-first (twelve-factor
  services, containers, serverless) and you want it typed into a struct.
- You value a dependency that will not churn — feature-complete and zero-dep.
- You want secret-friendly ergonomics (`,file`, `,unset`) without extra libraries.

**Avoid when:**

- You need layered config (files + flags + env + remote) — reach for a full config
  library instead of bolting layers onto this one.
- You want validation, hot-reload, or schema documentation built in.
- You dislike reflection-and-tags config and prefer explicit, compile-checked
  wiring by hand.

## Alternatives

- `sethvargo/go-envconfig` — closest peer; env-into-struct with mutators, a
  `Lookuper` abstraction, and context support. Use when you want testable env
  sources and per-value transformation hooks.
- `kelseyhightower/envconfig` — the older prefix-convention parser. Use when you
  prefer deriving variable names from field names over explicit tags, and don't
  need active maintenance.
- `spf13/viper` — full configuration stack (files, flags, env, remote KV, live
  reload). Use when env vars are only one of several sources you must merge.
- `ilyakaznacheev/cleanenv` — env plus file formats plus defaults plus a
  validation hook. Use when you want parsing and validation in one dependency.
- `joho/godotenv` — loads `.env` files into the process environment; complementary
  rather than competing. Use it alongside `env` for local development.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-07-28 | Repository created; reflection-based env-to-struct parsing[^4]. |
| v6–v9 | 2019–2022 | Major bumps for breaking API changes; module path versioning. |
| generics era | 2022+ | `ParseAs[T]` / `ParseAsWithOptions[T]` added after Go 1.18 generics. |
| v11 | current | Latest major line; maintainer declares the project feature-complete[^2]. |

## References

[^1]: caarlos0/env README — supported types, tags, options, and the unexported-field caveat. https://github.com/caarlos0/env
[^2]: caarlos0/env README, "Current state" — feature-complete, bug fixes only. https://github.com/caarlos0/env#current-state
[^3]: "Used and supported by" — Encore. https://github.com/caarlos0/env
[^4]: GitHub repository metadata — created 2015-07-28. https://github.com/caarlos0/env

## Tags

go, golang, configuration, environment-variables, struct-tags, reflection, twelve-factor, zero-dependency, config-parsing, library
