# peterbourgon/ff

> A flags-first configuration library for Go, layering env vars and config files on top of the standard `flag` package.

[GitHub repo](https://github.com/peterbourgon/ff) ·
[License: Apache-2.0](https://github.com/peterbourgon/ff/blob/main/LICENSE)

## Overview

`ff` (for "flags-first") is a small Go library from Peter Bourgon, author of
Go kit[^1]. Its guiding thesis is a single opinion: `myprogram -h` should always
print the complete configuration surface of a program, so every config parameter
should be defined as a flag. Given that, `ff` provides one thing — a `Parse`
function that fills those flags from three ordered sources: command-line args
(highest priority), environment variables, then config files (lowest)[^2].

The defining tension is scope discipline. `ff` deliberately refuses to be a
configuration framework. It has no struct tags, no reflection-based binding, no
`Get("key")` runtime lookups, no live reload, no validation DSL. Where
`spf13/viper` builds a global settings registry, `ff` binds directly to the
`*string`/`*int`/`*bool` pointers that `flag.FlagSet` already gives you. This
makes it trivial to reason about and hard to outgrow cleanly: everything is a
flag, so config that doesn't map to a flag (nested objects, dynamic keys, secrets
managers) has no home here.

The repo carries two generations at once. The stable v3 line wraps the standard
`flag.FlagSet` verbatim. The v4 line (long pre-release as of 2026) adds `ff.FlagSet`
with getopts-style short/long names, an `ff.Command` subcommand tree, and the
`ffhelp` package for help text[^3]. Both live in the same repo under Go module
version suffixes.

## Getting Started

```bash
go get github.com/peterbourgon/ff/v4
# stable line: go get github.com/peterbourgon/ff/v3
```

```go
fs := flag.NewFlagSet("myprogram", flag.ContinueOnError)
var (
	listen = fs.String("listen", "localhost:8080", "listen address")
	debug  = fs.Bool("debug", false, "log debug information")
	_      = fs.String("config", "", "config file (optional)")
)

// Fills flags from args, then env vars, then the config file — in that order.
err := ff.Parse(fs, os.Args[1:],
	ff.WithEnvVarPrefix("MY_PROGRAM"),        // -> MY_PROGRAM_LISTEN, MY_PROGRAM_DEBUG
	ff.WithConfigFileFlag("config"),
	ff.WithConfigFileParser(ff.PlainParser),  // whitespace "key value" lines
)
if err != nil {
	fmt.Fprintf(os.Stderr, "error: %v\n", err)
	os.Exit(1)
}
fmt.Printf("listen=%s debug=%v\n", *listen, *debug)
```

A flag set to `-listen=:9090` wins over `MY_PROGRAM_LISTEN` in the environment,
which wins over `listen :9090` in the config file.

## Architecture / How It Works

`ff.Parse` is the whole engine. It first delegates to the underlying flag set's
own `Parse` for command-line args, then walks the remaining unset flags looking
for matching environment variables (name uppercased, dashes to underscores,
prefixed if `WithEnvVarPrefix` is set), then reads the config file named by the
flag passed to `WithConfigFileFlag` and applies any values still unset. Because
priority is enforced by "already set wins," the order of the three passes is the
order of precedence — args, then env, then file[^2].

Config file formats are pluggable via `WithConfigFileParser`, which takes any
`func(io.Reader, func(name, value string) error) error`. `ff.PlainParser` (a
whitespace-separated `key value` format) ships in core. JSON, YAML, and TOML
parsers live in separate sub-packages (`ff/ffjson`, `ff/ffyaml`, `ff/fftoml`) so
that the core module pulls in no third-party dependencies — you opt into a YAML
parser only if you import one.

The v4 line adds `ff.FlagSet`, a hand-written flag container that supports
getopts(3) conventions the standard library never did: single-dash short flags
(`-v`), double-dash long flags (`--verbose`), combined shorts (`-abc`), and
repeatable/slice flags. `ff.Command` composes these into a tree via
`SetParent`, where a child flag set can parse its parents' flags too, and each
command holds an `Exec func(context.Context, []string) error`. `ffhelp` renders
usage from a flag set's reflected structure. The design keeps command parsing
orthogonal to presentation: no built-in colors, no tab-completion, no output
helpers — those are left to the caller.

## Production Notes

- **v4 has been pre-release for a long time.** The README explicitly steers
  production users to v3[^3]. v4's API (especially `ff.FlagSet` and
  `ff.Command`) has shifted across pre-release tags; pin an exact version and
  read the changelog before upgrading rather than tracking `@latest`.
- **Precedence is only across sources, not within a config file.** `ff` will not
  merge multiple config files or do deep-merge semantics. One file, one pass.
  Layered environments (base + override) are the caller's problem.
- **No secrets story.** Everything visible to `-h` is, by design, visible. `ff`
  is a poor fit for pulling secrets from Vault/SSM/Secret Manager; those don't
  map to a flag surface and you'll bolt them on separately.
- **Env var naming is mechanical, not configurable per-flag in v3.** The
  transform is `PREFIX_FLAG_NAME`. Flags with characters that don't map cleanly
  to env var names need care; verify the generated names rather than assuming.
- **Error handling is yours.** Unlike `flag.FlagSet`, the v4 `ff.FlagSet` does
  not print help to stderr as a parse side effect — you must check the returned
  error and render `ffhelp` yourself. This is more correct but trips people
  migrating from stdlib muscle memory.
- **Slice/repeatable flags are v4-only.** If you need repeatable flags on the
  stable line, you supply your own `flag.Value`.

## When to Use / When Not

**Use when:**
- You want stdlib `flag` semantics plus env vars and a config file, with no
  framework buy-in and (on the core module) zero external dependencies.
- Your configuration genuinely is a flat set of flags and you want `-h` to be the
  authoritative documentation.
- You're building a `kubectl`/`docker`-style subcommand CLI and want a light,
  declarative tree (v4 `ff.Command`).

**Avoid when:**
- You need nested/dynamic config, live reload, remote config stores, or
  reflection-based struct binding — reach for `viper` or hand-rolled loading.
- You want batteries-included CLI ergonomics (completion, colored help,
  generated docs) out of the box — `cobra` covers that.
- You require a stable, frozen API today and don't want to sit on a pre-release —
  stay on v3 and accept its narrower feature set.

## Alternatives

- spf13/cobra — use instead when you want a full CLI framework with generated
  help, shell completion, and docs, and don't mind the heavier surface.
- spf13/viper — use instead when config lives in many layered sources (remote
  KV, live-reloaded files, nested keys) rather than a flat flag surface.
- urfave/cli — use instead when you want a mature, opinionated command/flag API
  in one package with less assembly than `ff.Command`.
- alecthomas/kingpin — use instead when you prefer a fluent builder DSL for flags
  and commands.
- spf13/pflag — use instead when you only need POSIX/getopts flag parsing to drop
  under cobra, without the env/file layering `ff` adds.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-08-31 | Repo created; `ff.Parse` over `flag.FlagSet` with env + file sources[^4]. |
| v2 | ~2019 | Module path bump; parser options refined. |
| v3 | ~2021 | Stable line; recommended for production, pluggable config parsers[^3]. |
| v4 | pre-release | Adds `ff.FlagSet` (short/long flags), `ff.Command`, `ffhelp`; still pre-release as of 2026[^3]. |

## References

[^1]: Peter Bourgon — author of Go kit and other Go libraries. https://peter.bourgon.org/
[^2]: `ff` README, "Parse priority" — args > env vars > config files. https://github.com/peterbourgon/ff#parse-priority
[^3]: `ff` README note on v4 pre-release status and v3 stability. https://github.com/peterbourgon/ff#note
[^4]: GitHub repository metadata (created 2017-08-31), fetched 2026-07. https://github.com/peterbourgon/ff

## Tags

go, cli, configuration, flags, environment-variables, config-files, command-line, getopts, library, argument-parsing
