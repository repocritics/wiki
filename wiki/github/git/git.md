# git/git

> The distributed version control system that won — Linus Torvalds's "stupid content tracker," now the substrate under nearly all software development.

[GitHub repo](https://github.com/git/git) ·
[Official website](https://git-scm.com) ·
License: GPL-2.0-only (with some files under compatible licenses)[^1]

## Overview

Git is a distributed version control system written primarily in C. Linus Torvalds wrote the first version in April 2005 over roughly two weeks, after the BitKeeper license that the Linux kernel had relied on was withdrawn[^2]. Junio Hamano took over as maintainer later that year and still holds the role in 2026. This GitHub repository is explicitly a **publish-only mirror**: real development happens on the `git@vger.kernel.org` mailing list via emailed patches, not through GitHub pull requests[^3]. The repo's own description tells you to route contributions through GitGitGadget, which converts PRs into mailing-list patches.

The defining idea is that Git is a **content-addressable filesystem with a VCS UI bolted on top**. Every object — blob, tree, commit, tag — is stored under the SHA-1 (now transitioning to SHA-256) hash of its contents. A commit is an immutable snapshot of the whole tree plus parent pointers, not a diff. Branches are just movable pointers to commits; the entire history is a directed acyclic graph. This model is why Git operations are mostly local and fast, why merging is cheap, and why the tool is simultaneously trusted and famously hard to reason about.

The enduring tension is **model simplicity vs. interface complexity**. The object model is small and elegant; the porcelain (`git`'s user-facing commands) accreted over 20 years and exposes plumbing concepts (index/staging area, detached HEAD, reflog, the difference between `reset --soft/--mixed/--hard`) that regularly defeat experienced engineers. Git is the near-universal default not because it is easy but because the ecosystem (GitHub, GitLab, CI, every IDE) standardized on it.

## Getting Started

```bash
# macOS (Homebrew) / Debian-Ubuntu / Fedora
brew install git
sudo apt install git
sudo dnf install git
```

```bash
git init my-project && cd my-project
git config user.name "Your Name"
git config user.email "you@example.com"

echo "hello" > README.md
git add README.md
git commit -m "Initial commit"

git switch -c feature      # create + switch to a branch
# ... edit files ...
git add -p                 # stage hunks interactively
git commit -m "Add feature"
git switch main
git merge feature
```

```bash
# Inspect the content-addressable store directly
git cat-file -p HEAD^{tree}   # list the tree object HEAD points at
git rev-parse HEAD            # resolve HEAD to its full object hash
```

## Architecture / How It Works

Git's storage is four object types in `.git/objects`, each keyed by the hash of its zlib-compressed content:

- **blob** — file contents (no filename, no metadata).
- **tree** — a directory listing mapping names → blob/tree hashes + mode bits.
- **commit** — a tree hash, zero or more parent hashes, author/committer, message.
- **tag** — an annotated, signable pointer to another object.

Because objects are addressed by content hash, identical content is stored once, and any change anywhere ripples new hashes up to the root commit. This is what makes history tamper-evident and why rewriting history (`rebase`, `filter-repo`) necessarily changes every downstream commit hash.

The **index** (a.k.a. staging area, `.git/index`) is a binary file holding the proposed next tree. It is the concept most tutorials skip and most users trip on: `git add` writes to the index, `git commit` snapshots the index, and `git status` compares working tree ↔ index ↔ HEAD. Understanding these three trees explains nearly every confusing Git behavior.

Objects start **loose** (one file each), then get compacted into **packfiles** — delta-compressed archives with an index — by `git gc` or during network transfer. Refs (`.git/refs`, `packed-refs`) are the mutable layer: branches, tags, and remotes are just files containing a hash. HEAD is a symref to the current branch. The **reflog** records every ref movement locally, which is why "lost" commits after a bad reset are usually recoverable.

Two long-running platform shifts matter in 2026: the **SHA-1 → SHA-256** object-format migration (SHA-256 repos are usable but interop with SHA-1 hosting is still limited), and a multi-year **C-to-Rust** effort — the project accepts Rust as an optional build dependency and has begun landing Rust components, though the core remains C[^4]. Git is also single-binary with subcommands dispatched to built-ins or `git-<cmd>` executables on `PATH`, which is how third-party subcommands (`git-lfs`, `git-absorb`) integrate transparently.

## Production Notes

**Monorepo scaling is the real operator pain.** Vanilla Git degrades on very large repos and working trees. Mitigations, mostly contributed by Microsoft and GitHub: **partial clone** (`--filter=blob:none` to fetch objects lazily), **sparse-checkout** + sparse-index (materialize only part of the tree), the **commit-graph** file (fast history traversal), and **`fsmonitor`** (filesystem watcher to avoid scanning millions of files on `git status`)[^5]. Scalar, now bundled with Git, wires these together with sane defaults.

**Line endings and case sensitivity bite cross-platform teams.** `core.autocrlf` misconfiguration produces spurious whole-file diffs; commit a `.gitattributes` with explicit `text`/`binary` rules rather than relying on per-machine config. On macOS/Windows the default filesystem is case-insensitive, so `File.txt` and `file.txt` collide — a class of bug that never reproduces on Linux CI.

**History rewriting is a footgun with a blast radius.** `git rebase` / `git filter-repo` change commit hashes; force-pushing a shared branch invalidates everyone else's clone and can silently drop others' work. Prefer `--force-with-lease` over `--force`. `BFG` and `git filter-repo` (not the deprecated `filter-branch`) are the tools for purging secrets/large files, but the old objects survive in every existing clone and in hosting-provider caches until forced-GC'd — treat any leaked secret as compromised regardless.

**Large binaries do not belong in Git.** Every version of a big file is stored forever in history and bloats every clone. Use `git-lfs` (pointer files + separate object store) or `git-annex`. Retrofitting LFS onto an existing repo requires history rewriting.

**Submodules are widely regarded as the worst part.** They work but have sharp edges (detached HEADs, easy-to-forget `--recurse-submodules`, pinned-commit drift). Many teams migrate to subtree merges or a monorepo instead.

**Signing and supply chain.** Commit/tag signing supports GPG, S/MIME, and SSH keys (`gpg.format = ssh`); `git verify-commit` and allowed-signers files enable provenance checks. This is increasingly required in regulated pipelines.

## When to Use / When Not

**Use when:**
- You are versioning source code or any text-heavy, mergeable content — this is the default and correct choice.
- You need offline-capable, distributed history where every clone is a full backup.
- You want ecosystem gravity: hosting, CI, review tooling, and IDE support all assume Git.

**Avoid / augment when:**
- You are versioning large binaries or game/media assets — pair with LFS, or consider Perforce/Plastic SCM (Unity Version Control) which are built for that.
- You have a giant monorepo and can't invest in partial-clone/sparse tooling — evaluate Scalar first, or a purpose-built VCS.
- Your users are non-technical and need a linear, lock-based workflow — Git's flexibility is overhead they will fight.

## Alternatives

- martinvonz/jj — Jujutsu, a Git-compatible VCS with a cleaner model (no staging area, every change is a commit, first-class undo); use when you want Git interop but hate the porcelain.
- mercurial/mercurial — Mercurial (Hg); use when you want a more consistent CLI and are in an ecosystem (historically Facebook/Meta) that standardized on it.
- Perforce Helix Core — use for very large binary/game-asset repositories needing centralized file locking.
- Fossil (fossil-scm) — use when you want VCS + issue tracker + wiki in one self-contained SQLite-backed binary.
- Pijul — use when you want a theory-of-patches model (commutative patches, sound merges) rather than snapshot-based history.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2005-04 | Linus Torvalds writes Git after BitKeeper fallout[^2]. |
| maintainer | 2005-07 | Junio Hamano becomes and remains lead maintainer. |
| 1.0 | 2005-12-21 | First 1.0 release; core plumbing/porcelain stabilizes. |
| 1.5.0 | 2007-02 | Major UI overhaul; the modern porcelain most users know. |
| 2.0 | 2014-05 | Default `push.default` → `simple`; behavior cleanups. |
| 2.23 | 2019-08 | `git switch` / `git restore` split off from overloaded `checkout`. |
| 2.29 | 2020-10 | Experimental SHA-256 object format. |
| 2.38 | 2022-10 | Scalar bundled for large-repo workflows[^5]. |
| 2.x | 2024–2026 | Ongoing SHA-256 interop work and optional Rust components[^4]. |

## References

[^1]: Git is licensed GPL-2.0-only; the GitHub API reports the license as NOASSERTION because the tree mixes some files under other GPLv2-compatible licenses. See the repository `COPYING` and `LICENSES/` files. https://github.com/git/git/blob/master/COPYING
[^2]: Linus Torvalds, initial Git commit and 2005 kernel mailing-list announcement following the BitKeeper license withdrawal. https://git-scm.com/book/en/v2/Getting-Started-A-Short-History-of-Git
[^3]: Repository description and README: "This is a publish-only repository but pull requests can be turned into patches to the mailing list via GitGitGadget." https://github.com/git/git
[^4]: Git build system accepts an optional Rust toolchain; the project has begun integrating Rust components alongside the C core. https://git-scm.com/docs
[^5]: Microsoft/GitHub scaling work — partial clone, sparse-checkout, commit-graph, fsmonitor, and Scalar. https://github.blog/2022-10-13-the-story-of-scalar/

## Tags

version-control, git, distributed-vcs, c, content-addressable, source-control, devtools, cli, scm, monorepo
