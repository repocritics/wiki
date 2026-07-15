# lint-staged/lint-staged

> Runs linters, formatters, and other tasks against only the files staged in a git commit — not the whole tree.

[GitHub repo](https://github.com/lint-staged/lint-staged) ·
[npm package](https://www.npmjs.com/package/lint-staged) ·
[License: MIT](https://github.com/lint-staged/lint-staged/blob/main/LICENSE)

## Overview

lint-staged is a Node CLI that takes the list of files git has staged for the
next commit, filters them by glob pattern, and runs a configured command
(ESLint, Prettier, Stylelint, tests, anything on `$PATH`) against just that
subset. It was created by Andrey Okonetchnikov in 2016 and is the de facto way
JavaScript projects wire "format and lint on commit" without paying the cost of
linting the entire repository each time[^1].

The core idea is small, but the value is entirely in the edge cases it handles.
A naive `eslint $(git diff --name-only)` breaks the moment a file is *partially*
staged (some hunks staged, some not), because the linter's autofix would write
the unstaged changes into the commit too. lint-staged's real job is git
plumbing: it stashes unstaged content, runs tasks against the staged snapshot,
re-applies edits, restores the stash, and rolls everything back if a task
fails[^2]. That safety machinery — not the glob matching — is why the project
exists and why reimplementing it as a shell one-liner usually goes wrong.

lint-staged is a *task runner*, not a hook manager. It does not install the git
`pre-commit` hook itself; that is delegated to a separate tool, almost always
typicode/husky. This split is the most common source of setup confusion for
newcomers, who expect one package to do both.

## Getting Started

```bash
npm install --save-dev lint-staged husky
npx husky init          # creates .husky/ and a pre-commit hook
```

```bash
# .husky/pre-commit
npx lint-staged
```

```json
// package.json — or .lintstagedrc.json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{css,md,json}": "prettier --write"
  }
}
```

Each key is a picomatch glob; each value is a command (or array of commands run
in order). lint-staged appends the matched absolute file paths as arguments, so
`"*.js": "eslint --fix"` becomes `eslint --fix /abs/a.js /abs/b.js`. Commands
that only report errors abort the commit; commands that edit files have their
changes automatically re-staged into the commit.

## Architecture / How It Works

A single `lint-staged` run is a sequence of git operations wrapped around your
tasks:

1. **Resolve staged files** via `git diff --staged --diff-filter=ACMR` (added,
   copied, modified, renamed by default). The `--diff` / `--diff-filter` flags
   override this to lint arbitrary revision ranges instead.
2. **Back up** the working state with a `git stash`, preserving unstaged changes
   and (with `--hide-all`) untracked files. This stash is the recovery anchor:
   its hash is printed so you can `git apply --index` it manually if lint-staged
   is killed mid-run.
3. **Hide partially-staged changes.** For files that are only partly staged,
   the unstaged hunks are stashed away so tasks see exactly the staged content.
   This is the subtle correctness guarantee most hand-rolled scripts miss.
4. **Run tasks**, grouped by config file and glob. Tasks for different globs run
   concurrently by default (`--concurrent true`); commands chained in an array
   for one glob run serially. Rendering is handled by listr2[^3], and processes
   are spawned with tinyexec[^4].
5. **Re-apply and stage** any task edits, restore the hidden unstaged changes,
   and drop the stash. On any task failure the whole thing reverts to the
   original state (unless `--no-revert` / `--no-stash` opt out).

Configuration discovery is hierarchical: lint-staged searches upward from each
staged file for the nearest config (`package.json`, `.lintstagedrc*`,
`lint-staged.config.{js,mjs,cjs,ts}`), which is what makes it usable in a
monorepo — each package's files run against that package's config, in that
package's directory. JS config files may export a function receiving the matched
filenames, enabling dynamic command construction and per-file ignore filtering.

## Production Notes

- **Concurrency is a footgun when globs overlap.** Because tasks run in parallel
  by default, a config where `"*": "prettier --write"` and `"*.ts": "eslint
  --fix"` both match the same file will race two writers on that file. The fix is
  negation globs (`"!(*.ts)": "prettier --write"`) plus array chaining, not
  `--concurrent false` (which serializes everything and slows the common case).
- **The stash/restore dance can conflict.** If a task makes changes that don't
  cleanly re-apply against the restored unstaged hunks, you get a stash-apply
  conflict. This is rare but ugly, and it is the reason to keep tasks
  deterministic (avoid tools that rewrite unrelated formatting).
- **Husky is a separate dependency with its own version churn.** The
  lint-staged + husky pairing means two moving parts; husky's own major
  versions (v9 changed the hook file format) have historically broken setups
  independently of lint-staged.
- **Very large stages get chunked.** When the argument list exceeds the shell's
  limit, lint-staged splits the file list into multiple command invocations.
  Tasks that assume they see *all* files at once (e.g. a tool computing a global
  report) can misbehave; `--max-arg-length` tunes the threshold.
- **It only touches staged files, by design.** A broken file already committed
  will not be caught. lint-staged is a commit-time gate, not a substitute for a
  CI-wide `lint`/`format --check` job — teams that rely on it exclusively let
  bad state through via `--no-verify` or non-committing edits.
- **TypeScript config files need a modern Node.** `.lintstagedrc.ts` works only
  where Node can strip types (`--experimental-strip-types`, unflagged in Node
  v23.6.0); on older runtimes stick to `.mjs`/`.cjs`.

## When to Use / When Not

**Use when:**
- Your repo is JS/TS-centric and you want format-and-lint enforced at commit
  time without linting the whole tree.
- You need correct handling of partially-staged files and autofix re-staging.
- You have a monorepo and want per-package configs resolved automatically.

**Avoid when:**
- Your repo is polyglot and the hook logic isn't JS-centric — a
  language-agnostic framework fits better than a Node dependency.
- You want hooks and filtering in one tool with no Node runtime — a native git
  hook manager is lighter.
- You need repo-wide guarantees, not just staged files — enforce that in CI
  instead; lint-staged is bypassable with `git commit --no-verify`.

## Alternatives

- typicode/husky — not a replacement but the standard companion; it installs the
  `pre-commit` hook that invokes lint-staged. You generally need both.
- evilmartians/lefthook — a git hooks manager with built-in staged-file
  filtering and parallelism; use it when you want hooks + filtering in one
  fast, Node-optional binary.
- pre-commit/pre-commit — a Python-based, language-agnostic hook framework with
  a large plugin registry; use it for polyglot repos where a JS-only task runner
  is a poor fit.
- lint-staged/lint-staged.sh — a minimal shell script from the same org for the
  read-only case; use it when you only want to *check* staged files, never
  autofix them.
- nrwl/nx or turborepo — for monorepos where the real need is "run tasks against
  affected projects," not "against staged files"; different granularity.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2016-06-07 | First stable release[^1]. |
| 4.0.0 | 2017-06-18 | Concurrency and glob handling maturation. |
| 8.0.0 | 2018-10-29 | Config-file search, per-directory configs. |
| 10.0.0 | 2020-01-19 | Node baseline raised; partial-staging handling hardened. |
| 12.0.0 | 2021-11-13 | Continued Node modernization. |
| 13.0.0 | 2022-06-01 | Package moved to ESM. |
| 14.0.0 | 2023-08-13 | Node 16+ baseline. |
| 15.0.0 | 2023-10-14 | Further Node/tooling updates. |
| 16.0.0 | 2025-05-10 | Node baseline raised again. |
| 17.0.0 | 2026-05-06 | Latest major line. |

Repository ownership moved from the original `okonet/lint-staged` namespace to
the `lint-staged` GitHub org as the project grew beyond a single maintainer.

## References

[^1]: Andrey Okonetchnikov, "Make Linting Great Again" — 2016. https://medium.com/@okonetchnikov/make-linting-great-again-f3890e1ad6b8
[^2]: lint-staged README — command-line flags, stash/restore and `--no-stash` behavior. https://github.com/lint-staged/lint-staged#command-line-flags
[^3]: listr2 — terminal task-list renderer used by lint-staged. https://listr2.kilic.dev/
[^4]: tinyexec — process spawning library used by lint-staged. https://github.com/tinylibs/tinyexec

## Tags

javascript, git, git-hooks, linter, formatter, pre-commit, developer-experience, cli, code-quality, monorepo
