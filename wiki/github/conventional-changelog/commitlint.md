# conventional-changelog/commitlint

> A linter for Git commit messages that checks them against the Conventional Commits format (or a config you define).

[GitHub repo](https://github.com/conventional-changelog/commitlint) ·
[Official website](https://commitlint.js.org) ·
[License: MIT](https://github.com/conventional-changelog/commitlint/blob/master/license.md)

## Overview

commitlint parses a commit message and validates its structure against a set of rules — most commonly the Conventional Commits convention of `type(scope?): subject`[^1]. It is a Node.js monorepo (published as scoped `@commitlint/*` packages on npm) that grew out of the `conventional-changelog` ecosystem, where machine-readable commit history is the input that drives automatic changelog generation and semantic versioning. Enforcing the format at commit time is the upstream half of that pipeline: if commits are well-formed, tools like `semantic-release` and `conventional-changelog` can derive releases from them without human curation[^2].

The tool itself is small in ambition and deliberately so. It does not format code, run tests, or manage hooks — it reads a string, applies rules, and exits non-zero on failure. Everything else (when it runs, how the message is captured, what the rules are) is delegated: Git hooks are supplied by a separate tool such as Husky, and the rule set comes from a shareable config package. This unbundling is the project's defining design choice and the source of most of its setup friction — a working install is usually commitlint plus a config plus a hook manager, three moving parts that must agree.

The default rules live in `@commitlint/config-conventional`, based on the Angular commit convention, with types like `feat`, `fix`, `docs`, `chore`, `refactor`, `perf`, `test`, `build`, `ci`, `style`, and `revert`[^3]. Teams routinely extend or replace this. commitlint is considered stable and low-churn; the README explicitly frames it as a mature development tool rather than an actively expanding product[^4].

## Getting Started

```bash
# install the CLI and a shareable rule set
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

```js
// commitlint.config.js
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "scope-enum": [2, "always", ["api", "ui", "docs", "build"]],
  },
};
```

```bash
# lint the most recent commit
npx commitlint --from HEAD~1 --to HEAD --verbose

# lint a message from a file (how the commit-msg hook invokes it)
echo "fix(api): handle empty payload" | npx commitlint
```

To run automatically on every commit, wire it into a `commit-msg` Git hook via Husky:

```bash
npx husky init
echo 'npx --no -- commitlint --edit "$1"' > .husky/commit-msg
```

## Architecture / How It Works

commitlint is a monorepo of narrowly-scoped packages, each usable on its own:

- **`@commitlint/cli`** — the command-line entry point; parses flags and orchestrates the others.
- **`@commitlint/load`** — resolves and merges configuration (handles `extends`, plugins, and parser presets).
- **`@commitlint/read`** — pulls commit messages, either from a range (`--from`/`--to`), the edit file (`--edit`), or stdin.
- **`@commitlint/lint`** — the core: runs a parsed commit through the rule functions.
- **`@commitlint/format`** — renders the report (the colored problem/warning output).
- **`@commitlint/config-conventional`** and siblings (`config-angular`, `config-lerna-scopes`, `config-nx-scopes`, `config-workspace-scopes`) — shareable rule sets.

Configuration is discovered through cosmiconfig, so any of a long list of file names works — `.commitlintrc`, `.commitlintrc.{json,yaml,yml,js,cjs,mjs,ts,cts,mts}`, `commitlint.config.{js,cjs,mjs,ts,cts,mts}`, or a `commitlint` key in `package.json`[^5]. A message is first split into components (type, scope, subject, body, footer) by a conventional-commits parser, then each rule receives that parsed object plus its configured value. Rules are `[level, applicability, value]` tuples where level is `0` (off), `1` (warning), or `2` (error), and applicability is `"always"` or `"never"` — for example `"type-enum": [2, "always", [...]]` errors on any type outside the list, while `"header-max-length": [2, "always", 72]` caps the header. Custom rules and parser presets can be added via plugins.

Because commitlint only validates strings, it has no opinion about when it runs. In practice it is invoked from a Git `commit-msg` hook (message already written) or in CI against a range of pushed commits. The `--edit` flag reads the temporary file Git passes to the hook; the range mode is what CI uses to lint a whole PR.

## Production Notes

**The three-part setup is the main footgun.** commitlint, a config package, and a hook runner are separate installs, and a missing config produces the misleading `Please add rules to your commitlint.config.js` error even when the file exists — most often because the config could not be *loaded*, not because rules are absent.

**Node 24 changed module loading and broke JS config resolution in some layouts.** If a project has no `package.json` declaring module type, commitlint under Node 24+ can fail to load `commitlint.config.js` and emit the "add rules" error. The documented fixes are to add a `package.json` marking the project as ESM (`npm init es6`) or rename the config to `commitlint.config.mjs`[^6]. TypeScript configs (`.ts`) require a loader/runtime that can import them.

**Runtime floor moves with Node's LTS.** Current releases require Node.js `>= 22.12.0` and git `>= 2.13.2`[^7]; the minimum has been raised on major bumps, so pinning an old commitlint is common on legacy CI images. The maintainers state plainly that they are not a sponsored project and will not guarantee timely backported patches for old majors — security fixes for old lines are handled on a best-effort, PR-welcome basis[^8].

**CI vs local divergence.** Linting a range in CI (`--from`/`--to`) catches commits that bypassed local hooks (hooks are trivially skipped with `git commit --no-verify`), but merge commits, squashed histories, and rebases can produce headers that fail rules written for individual commits. Squash-merge workflows should generally lint the PR title instead of the constituent commits.

**Monorepo scope rules need generation.** The `config-nx-scopes` / `config-workspace-scopes` / `config-lerna-scopes` packages derive the allowed `scope-enum` from the workspace's project list at lint time, which means scope validation depends on the monorepo tool being installed and resolvable in the lint environment.

## When to Use / When Not

**Use when:**
- You want machine-parseable commit history to feed `semantic-release` or `conventional-changelog` for automated versioning and changelogs.
- You need to enforce a commit convention across a team or in CI, not just document it.
- You want a customizable rule engine rather than a fixed format — levels, enums, and plugins cover most policies.

**Avoid when:**
- You only want a convention *documented*, not enforced — a CONTRIBUTING note is lighter than three dependencies.
- Your team squash-merges everything and doesn't consume commit metadata downstream — the value largely evaporates.
- You want commit-message linting bundled with hook management and staged-file linting in one tool; commitlint is intentionally just the linter.

## Alternatives

- commitizen/cz-cli — interactive prompt that *guides* writing a conventional commit; complements rather than replaces commitlint (authoring vs. validation).
- conventional-changelog/commitlint's own `@commitlint/prompt` — use instead when you want the prompt and the linter from one project.
- semantic-release/semantic-release — use instead when your real goal is automated releases; it can be strict about commit format without a separate linter, though many pair the two.
- typicode/husky — not a linter but the hook layer commitlint usually rides on; use when you need to *run* commitlint (or anything) on Git events.
- gitleaks/gitleaks — different concern (secret scanning in commits); use when the thing you want to catch is credentials, not message format.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-02 | Repository created (originally `conventional-changelog-lint`)[^9]. |
| rename | ~2018 | Rebranded to `commitlint`, published under scoped `@commitlint/*` packages. |
| 17.0 | 2022-05 | Node 12 dropped; ESM/CJS dual publishing matured. |
| 18.0 | 2023-10 | Node 16 minimum; dependency modernization. |
| 19.0 | 2024-02 | Node 18 minimum; config resolution updates[^7]. |

## References

[^1]: Conventional Commits specification. https://www.conventionalcommits.org
[^2]: conventional-changelog monorepo. https://github.com/conventional-changelog/conventional-changelog
[^3]: `@commitlint/config-conventional` type-enum. https://github.com/conventional-changelog/commitlint/tree/master/%40commitlint/config-conventional
[^4]: commitlint README, "Roadmap" — project described as stable. https://github.com/conventional-changelog/commitlint#roadmap
[^5]: commitlint config file resolution (cosmiconfig). https://commitlint.js.org/reference/configuration.html
[^6]: commitlint README, "Important note about Node 24+". https://github.com/conventional-changelog/commitlint#config
[^7]: commitlint README, "Version Support and Releases". https://github.com/conventional-changelog/commitlint#version-support-and-releases
[^8]: commitlint README, "Releases" — no guaranteed backports for old majors. https://github.com/conventional-changelog/commitlint#releases
[^9]: GitHub repository metadata, created 2016-02-12. https://github.com/conventional-changelog/commitlint

## Tags

typescript, javascript, git, commit-conventions, linter, conventional-commits, cli, developer-tooling, monorepo, ci
