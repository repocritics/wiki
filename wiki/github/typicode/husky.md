# typicode/husky

> A thin wrapper that installs your project's Git hooks via `core.hooksPath`, so committing runs the checks you configure.

[GitHub repo](https://github.com/typicode/husky) ·
[Official website](https://typicode.github.io/husky) ·
[License: MIT](https://github.com/typicode/husky/blob/main/LICENSE)

## Overview

Husky is the most widely used way to run scripts on Git lifecycle events (pre-commit, commit-msg, pre-push, and the [other client-side hooks](https://git-scm.com/docs/githooks)) in JavaScript projects. Its entire job is small: put your hook scripts under a `.husky/` directory, point Git's `core.hooksPath` at them, and make that setup happen automatically when teammates run `npm install`. It does not lint, format, or test anything itself — it is the trigger, and you supply the commands (almost always [lint-staged](https://github.com/lint-staged/lint-staged) and/or [commitlint](https://github.com/conventional-changelog/commitlint)).

First published in 2014[^1], Husky was for years a heavier, config-driven tool: you declared hooks under a `husky` key in `package.json` and it wrote shims into `.git/hooks`. Version 5 (2021) was a ground-up rewrite that abandoned that model for `core.hooksPath` and plain shell files, and version 9 (2024) stripped the last of the boilerplate so hook files are now ordinary scripts[^2]. The current package is ~2 kB gzipped with no runtime dependencies[^3].

The defining tension is that Husky is a convenience layer over a Git mechanism that is itself trivial. Setting `git config core.hooksPath .husky` by hand is one command; Husky exists to make that survive `npm install`, degrade cleanly in CI and non-Git checkouts, and work across GUIs and shells. Most of its documented pain is not about hooks — it is about the `prepare`-script lifecycle it hooks into.

## Getting Started

```bash
npm install --save-dev husky
npx husky init
```

`husky init` creates `.husky/pre-commit` (containing `npm test`), sets `core.hooksPath`, and adds a `prepare` script to `package.json` so the setup re-runs on every `npm install`:

```json
{
  "scripts": {
    "prepare": "husky"
  }
}
```

A v9 hook file is just a script — no shebang or sourced helper required:

```sh
# .husky/pre-commit
npx lint-staged
```

```sh
# .husky/commit-msg  — validate the message with commitlint
npx --no -- commitlint --edit "$1"
```

## Architecture / How It Works

Husky's install step (`husky` / `husky init`) does two things: it runs `git config core.hooksPath .husky/_` and populates that hidden `.husky/_/` directory with generated wrapper scripts — one named for each supported hook, plus the shared `husky.sh` helper and a `.gitignore` that excludes the internals from version control. Your own hooks live one level up in `.husky/` and are committed.

When Git fires an event, it executes `.husky/_/<hookname>` (because that is where `core.hooksPath` points). That wrapper sources `husky.sh` and then invokes your `.husky/<hookname>` file. `husky.sh` is where the cross-environment logic lives: it honors `HUSKY=0` to disable hooks, sources an optional `~/.config/husky/init.sh` (used to put `node`/`npm` on `PATH` for GUI clients that don't inherit a login shell), and prints the deprecation warning if it detects legacy boilerplate in your hook file.

This indirection is the reason v9 hook files no longer need the two-line preamble (`#!/usr/bin/env sh` and `. "$(dirname -- "$0")/_/husky.sh"`) that v5–v8 required: the wrapper in `_/` now does the sourcing, so the user-facing file can be plain shell. Those two lines are deprecated in v9, still tolerated with a warning, and slated to fail in v10[^2].

Because Husky drives everything through the standard `prepare` npm lifecycle script, the install is opt-in per clone and requires no postinstall hackery — but it also inherits every quirk of when `prepare` does and doesn't run.

## Production Notes

**The `prepare`-in-CI footgun.** `prepare` runs after `npm install`, including on CI. If you install with `npm ci --omit=dev` (or `--production`), Husky is not present but `prepare` still fires, and the build dies with `husky: command not found`. The idiomatic guard in v9 is `"prepare": "husky || true"`, or a script that skips when `CI` is set or `.git` is absent. Husky already no-ops when there is no `.git` directory (e.g. installing the package as a dependency of someone else's project), but the missing-binary case is on you.

**Hooks silently don't run.** Common causes: Git older than 2.9 (which introduced `core.hooksPath`); a pre-existing `core.hooksPath` that someone else set; installing Husky in a sub-package instead of the repo root (the `prepare` script must run where `.git` lives — in monorepos this usually means installing Husky once at the root); or `git commit --no-verify` / `-n`, which bypasses all hooks by design. There is no error when hooks are simply skipped, which makes "it works on my machine" reports frequent.

**GUI and PATH issues.** GUI clients (Tower, SourceTree, GitHub Desktop, some editors) may launch hooks with a minimal `PATH` that lacks `node`, so a hook running `npx lint-staged` fails with a not-found error even though the terminal works. The supported fix is `~/.config/husky/init.sh` to export the right `PATH`. Windows works because Git for Windows ships a POSIX `sh`, but hook scripts must stay POSIX-shell-compatible.

**Migration cost is real.** The v4→v5+ change was not a version bump but a different tool: config moved out of `package.json` into committed shell files, and there is no automatic migration — you rewrite your hooks. Teams upgrading across that boundary should expect to touch every hook and their `prepare` script.

**Scope discipline.** Husky runs whatever you put in the file with no sandboxing, ordering guarantees, or parallelism. Slow pre-commit hooks are the top developer complaint about any hook manager; keep pre-commit work staged-files-only (via lint-staged) and push heavier checks to pre-push or CI.

## When to Use / When Not

**Use when:**
- Your project is Node/npm-centric and you want hooks that install themselves for the whole team on `npm install`.
- You want plain shell hook files under version control, not a config DSL.
- You're pairing it with lint-staged and/or commitlint — the mainstream setup.

**Avoid when:**
- The repo is polyglot and non-Node contributors won't have npm — a language-agnostic manager fits better.
- You need parallel hook execution, per-hook environments, or pinned tool versions out of the box.
- You want zero install-lifecycle surface area; a one-line `core.hooksPath` in your onboarding docs may be enough.

## Alternatives

- pre-commit/pre-commit — Python-based, language-agnostic framework that manages hook tool versions in isolated environments; use it when the repo is polyglot or you want reproducible, pinned linters.
- evilmartians/lefthook — single Go binary with YAML config and parallel execution; use it when hook speed or non-Node teams matter.
- toplenboren/simple-git-hooks — minimal `core.hooksPath` setup configured from `package.json`; use it when you want Husky's mechanism with less surface.
- lint-staged/lint-staged — complementary, not a replacement: runs commands against staged files. Husky triggers it.
- conventional-changelog/commitlint — complementary: the usual `commit-msg` payload for enforcing message conventions.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014 | Initial release; wrote shims into `.git/hooks`[^1]. |
| 4.0 | 2019-11 | Config under the `husky` key in `package.json`; peak "config-driven" era. |
| 5.0 | 2021-01 | Full rewrite: `core.hooksPath` + `.husky/` shell files; released to sponsors first[^4]. |
| 6.0 | 2021-04 | Simplified install (`husky install` + `husky add`), lighter helper. |
| 7.0 | 2021-09 | Maintenance and size reduction. |
| 8.0 | 2022-05 | Stability release on the v6 model. |
| 9.0 | 2024-01 | Boilerplate-free hooks, `husky init`, ~2 kB, no deps; legacy preamble deprecated[^2]. |

## References

[^1]: Package history, npm — husky. https://www.npmjs.com/package/husky?activeTab=versions
[^2]: Husky documentation, "Migrate from v4 to v9" and hook format notes. https://typicode.github.io/husky/migrate-from-v4.html
[^3]: Repository README — "Just 2 kB gzipped with no dependencies." https://github.com/typicode/husky
[^4]: Husky v9.0.1 release / changelog (links back through the v5 sponsorware release). https://github.com/typicode/husky/releases/tag/v9.0.1

## Tags

javascript, git, git-hooks, pre-commit, commit-msg, developer-tools, cli, linting, npm, monorepo
