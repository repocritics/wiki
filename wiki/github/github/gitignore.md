# github/gitignore

GitHub's curated collection of `.gitignore` templates — the canonical "what files should I ignore for Project Type X" reference.

## What it is

A directory of `.gitignore` template files, one per common language, framework, IDE, and toolchain. When you initialize a new repository on GitHub.com and pick "Add .gitignore", you're picking from this list. The templates are community-contributed and maintained by GitHub's docs team; the repo doubles as the source-of-truth for what ships in the GitHub web UI.

## Key features

- Three top-level categories: language/framework templates (`Python.gitignore`, `Node.gitignore`, etc.), global templates for editors/OSes (`Global/Vim.gitignore`, `Global/macOS.gitignore`), and community templates in `community/`.
- Each template is a standalone file you can drop straight into a project root.
- Used as the source list for GitHub.com's "Add .gitignore" picker on new repos.
- CC0-1.0 licensed — the most permissive license possible, so derivative IDE templates and `gh repo create` tooling can freely vendor or reference them.

## Tech stack

- Plain text `.gitignore` files. No code, no build system.
- Markdown documentation per category.

## When to reach for it

- You're starting a new project and want a vetted baseline `.gitignore` for your stack rather than writing one from scratch.
- You're auditing what to ignore in an existing project against the canonical reference.
- You're building tooling (project generators, scaffolders) that needs a license-clean source of starter `.gitignore` content.

## When *not* to reach for it

- You want a single one-size-fits-all `.gitignore` — the value here is the per-language curation.
- You want per-project enforcement of "must ignore X" rules — that's lint territory (e.g. pre-commit hooks), not template territory.

## Maturity signal

174k stars, 82k forks — the high fork count reflects how often individuals vendor templates into their own repos. 14-year-old project under GitHub's stewardship with CC0-1.0 license. Open-issues count of 64 is unusually low, signaling tight triage. Last push 2026-05-21.

## Alternatives

- `gitignore.io` (Toptal) — use when you want a web UI that composes multiple language templates into one file.
- IDE-bundled `.gitignore` (JetBrains, VS Code project templates) — use when you want IDE-managed defaults.

## Notes

CC0-1.0 is the rare "no rights reserved" license — these templates can be vendored into proprietary tooling without attribution. The repo is a frequent target for first-time OSS contributions because the "add language X" PRs are well-scoped and the maintainers' review bar is clear.

## Tags

git, gitignore, awesome-list, developer-tools, templates, creative-commons, configuration
