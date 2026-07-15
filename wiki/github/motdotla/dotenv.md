# motdotla/dotenv

> Zero-dependency Node.js module that loads variables from a `.env` file into `process.env`.

[GitHub repo](https://github.com/motdotla/dotenv) ·
[Official website](https://www.dotenv.org) ·
[License: BSD-2-Clause](https://github.com/motdotla/dotenv/blob/master/LICENSE)

## Overview

dotenv reads a `.env` file, parses each `KEY=value` line, and assigns the result to `process.env` at runtime. It is a direct descendant of the Ruby `dotenv` gem and an implementation of the config principle from The Twelve-Factor App: keep configuration in the environment, separate from code[^1]. The scope is deliberately tiny — it has no runtime dependencies and the core does one thing, which is why it sits under a large fraction of the Node ecosystem as an indirect dependency.

The library is best understood by what it does *not* do. Core dotenv does not expand variables (`${OTHER_VAR}`), does not run command substitution, does not encrypt secrets, and does not manage multiple environments. Each of those was historically either a separate companion package (`dotenv-expand`) or, since roughly 2024, folded into the maintainer's successor project, dotenvx[^2]. The README now leads with a banner promoting dotenvx, and most of the "Advanced" section delegates to it. This is the defining tension around dotenv today: the package is stable and near-universal, but its own author is steering new work toward a different, more featureful tool — so "use dotenv" increasingly means "use the minimal loader and reach for dotenvx (or your platform's secret manager) for anything beyond that."

## Getting Started

```sh
npm install dotenv --save
```

```ini
# .env
HELLO="Dotenv"
OPENAI_API_KEY="your-api-key-goes-here"
```

```javascript
// index.js — load as early as possible, before any module reads process.env
require('dotenv').config()
console.log(`Hello ${process.env.HELLO}`)

// ESM equivalent: import 'dotenv/config'
```

The `config()` call is synchronous and returns `{ parsed }` on success or `{ error }` on failure. The pure parser is also exported for arbitrary buffers: `dotenv.parse(Buffer.from('BASIC=basic'))` returns `{ BASIC: 'basic' }`.

## Architecture / How It Works

The whole library is a few hundred lines. `config()` resolves a path (default `path.resolve(process.cwd(), '.env')`), reads the file synchronously with `fs.readFileSync`, hands the contents to `parse()`, then calls `populate()` to write keys onto a target object (`process.env` by default, overridable via the `processEnv` option).

The parser is a single regular expression that walks the file line by line. Its rules matter in practice: lines starting with `#` are comments; an unquoted `#` mid-value begins an inline comment (so a value containing `#` must be quoted — this became a breaking change in v15); whitespace is trimmed from unquoted values but preserved inside quotes; single, double, and backtick quotes are all recognized; and double-quoted values expand `\n` into real newlines. Multiline values spanning actual line breaks are supported since v15 as well.

Key design decisions with downstream consequences:

- **First value wins, existing env is never overwritten.** If a key already exists in `process.env`, dotenv leaves it alone unless you pass `override: true`. With an array of paths, files are merged left-to-right and the first definition wins (last wins under `override`). This is what makes real shell/CI environment variables authoritative over the `.env` file.
- **No expansion in core.** `DATABASE_URL="postgres://${USER}@host"` is stored literally. You must layer `dotenv-expand` on top, or use dotenvx.
- **Import-order sensitivity.** Because `config()` mutates a global at call time, any module that reads `process.env` during its own top-level initialization must be imported *after* dotenv runs. Under ESM this is a common footgun; the fix is `import 'dotenv/config'` (or a tiny `load-env.mjs` wrapper) placed before other imports, or the `dotenv run --` CLI added in the v17 line.

## Production Notes

- **It is a development convenience, not a secret manager.** In real deployments the standard advice is to set variables through the platform (systemd, Kubernetes secrets, the PaaS config UI) and let dotenv be a no-op because the vars already exist. Do not commit an unencrypted `.env` — if you need secrets in git, that is the specific problem dotenvx encryption exists to solve.
- **The v17 startup log line surprised people.** Recent versions print a message like `injected env (2) from .env` to stdout/stderr on every `config()` call. This broke scripts that parse process output and annoyed library authors whose transitive use of dotenv started emitting noise. The escape hatch is the `quiet: true` option (or `DOTENV_CONFIG_QUIET=true`); budget a one-line change when upgrading into v17.
- **Front-end builds do not work the way people expect.** dotenv runs in Node, not the browser. Under Webpack/React there is no `fs` and often no `process`, so values must be injected at build time via `DefinePlugin` or a wrapper like `dotenv-webpack`. Create React App only exposes vars prefixed `REACT_APP_`; Next.js, Vite, etc. each have their own client-exposure rules. Importing dotenv directly into client code is a common mistake.
- **`.env.vault` is a dead end.** An encrypted-vault feature shipped in the v16 era but has effectively been superseded by dotenvx; new projects should not adopt `.env.vault`.
- **Node now has a built-in.** Node 20.6+ ships `--env-file=.env` (and `process.loadEnvFile`). For simple loading on modern Node you may not need the dependency at all, though the built-in's parsing and multi-file/override semantics differ from dotenv's.

## When to Use / When Not

**Use when:**
- You want a stable, zero-dependency way to load local/dev config into `process.env`.
- You are on Node older than 20.6, or need dotenv's specific parsing/override/multi-file behavior.
- You want the smallest possible surface and will handle secrets/expansion elsewhere.

**Avoid (or supplement) when:**
- You need variable expansion, command substitution, encryption, or committed-to-git secrets — reach for dotenvx.
- You are configuring a browser bundle — use your bundler/framework's env mechanism instead.
- You are on modern Node and only need basic loading — the built-in `--env-file` may remove the dependency.
- You need a real secrets backend (Vault, AWS/GCP Secrets Manager, Doppler) for production rotation and audit.

## Alternatives

- dotenvx/dotenvx — successor by the same author; use when you need encryption, `${VAR}` expansion, command substitution, or multi-environment `.env` files.
- nodejs/node (`--env-file` / `process.loadEnvFile`) — use when you're on Node 20.6+ and only need to load a plain `.env` with no extra features.
- motdotla/dotenv-expand — use when you want to stay on core dotenv but need `${VAR}` interpolation layered on top.
- mrsteele/dotenv-webpack — use when you need `.env` values injected into a Webpack front-end bundle at build time.
- toddbluhm/env-cmd — use when you'd rather set env from files via a CLI wrapper around your run command than call `config()` in code.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-07-05 | First commit; Node port of the Ruby dotenv gem[^1]. |
| 8.0.0 | 2019-08 | Long-lived 8.x line; the widely-pinned "classic" era. |
| 9.0.0 | 2021-05-05 | Dropped older Node support; parser and API cleanups. |
| 10.0.0 | 2021-05-20 | Maintenance major. |
| 15.0.0 | 2022-01-31 | Multiline values; inline `#` comment parsing became a breaking change. |
| 16.0.0 | 2022-02-02 | Multi-path arrays, `override`/`processEnv` options; `.env.vault` explored later in the line. |
| 17.0.0 | 2025-06-27 | Bundled CLI (`dotenv run --`); default startup log line, tunable via `quiet`. |
| 17.4.2 | 2026-04-12 | Current release at time of writing. |

## References

[^1]: The Twelve-Factor App, "Config." dotenv implements storing config in the environment. https://12factor.net/config
[^2]: dotenvx — encryption, variable expansion, and multi-environment support, positioned by dotenv's author as its successor. https://dotenvx.com

## Tags

javascript, nodejs, environment-variables, configuration, dotenv, secrets-management, twelve-factor, zero-dependency, config-loader, cli
