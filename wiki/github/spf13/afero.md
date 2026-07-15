# spf13/afero

> A filesystem abstraction for Go: one `afero.Fs` interface, swappable backends (real disk, in-memory, archive, cloud), and a drop-in mirror of the `os` package.

[GitHub repo](https://github.com/spf13/afero) ·
[License: Apache-2.0](https://github.com/spf13/afero/blob/master/LICENSE.txt)

## Overview

Afero is a Go library that puts an interface (`afero.Fs`) between your code and
the filesystem, so the same code can run against the real OS, an in-memory
filesystem, a ZIP/TAR archive, or a remote backend without changes[^1]. It was
written by Steve Francia (spf13), and its practical importance comes from being
the filesystem layer under his more widely used tools — Hugo and Viper both
depend on Afero for file access and config loading[^2]. That lineage means Afero
is battle-tested in real programs even though it is rarely the headline
dependency.

The core value proposition is testability. Code that calls `os.ReadFile`
directly is hard to unit-test without touching disk; code that accepts an
`afero.Fs` can be handed a `MemMapFs` in tests and run entirely in memory with
no fixtures to clean up. The second benefit, backend portability, is real but
narrower in practice: most projects use exactly two backends (OsFs in
production, MemMapFs in tests) and never touch the cloud adapters.

The defining tension is that Afero predates and now overlaps with Go's own
`io/fs` package (Go 1.16, 2021)[^3]. `io/fs` standardized a **read-only**
filesystem interface into the standard library, covering a large fraction of
what people historically reached for Afero to do. Afero's remaining
justification is write support, mutation testing (rename, concurrent writes),
and filesystem composition — the read-only case is now better served by the
standard library.

## Getting Started

```bash
go get github.com/spf13/afero
```

The pattern is: accept `afero.Fs` in your functions, inject `OsFs` in
production, inject `MemMapFs` in tests.

```go
package config

import "github.com/spf13/afero"

// Accept the interface, not the concrete OS.
func Load(fs afero.Fs, path string) ([]byte, error) {
    return afero.ReadFile(fs, path) // afero.* helpers mirror os/ioutil
}
```

```go
func TestLoad(t *testing.T) {
    fs := afero.NewMemMapFs() // fast, isolated, nothing to clean up
    afero.WriteFile(fs, "/etc/app.conf", []byte("ok"), 0o644)

    got, err := Load(fs, "/etc/app.conf")
    if err != nil || string(got) != "ok" {
        t.Fatalf("got %q err %v", got, err)
    }
}
```

## Architecture / How It Works

Afero is two interfaces plus a set of implementations. `afero.Fs` describes a
filesystem (Create, Open, Mkdir, Remove, Rename, Stat, etc.); `afero.File`
describes an open file (Read, Write, Seek, Readdir). Everything else is either a
backend that implements `Fs` or a free function (`afero.ReadFile`,
`afero.WriteFile`, `afero.Exists`, `afero.WalkDir`) that operates on any `Fs`.

The backends fall into three groups:

- **Concrete stores.** `OsFs` forwards every call to the `os` package. `MemMapFs`
  is a concurrency-safe in-memory tree guarded by locks; it is the reason Afero
  exists for most users.
- **Composers** — filesystems that wrap another `Fs`. `CopyOnWriteFs(base,
  overlay)` reads through to a base and diverts writes to an overlay;
  `CacheOnReadFs(base, cache, ttl)` lazily copies files from a slow base into a
  fast cache on first read; `BasePathFs(src, dir)` re-roots all paths into a
  subdirectory (a chroot/jail); `ReadOnlyFs` and `RegexpFs` restrict what passes
  through. These compose — a copy-on-write over a read-only OS over a memory
  overlay is a normal construction.
- **Adapters.** `zipfs`/`tarfs` expose an archive as a read-only `Fs`; `HttpFs`
  wraps an `Fs` for `http.FileServer`; `NewIOFS` presents an Afero filesystem as
  a standard-library `fs.FS`, and `FromIOFS` goes the other way (wrapping an
  `embed.FS` or any `fs.FS`, read-only).

The composition model is Afero's most distinctive feature and its most
underused one. The seams are clean because every layer speaks the same
interface, but the layered filesystems (`CacheOnReadFs` in particular) are
where behavior gets subtle — cache invalidation and TTL semantics are the
overlay's business, not the base's.

## Production Notes

- **MemMapFs is not a byte-identical OS emulator.** Its permission handling,
  path normalization, symlink behavior, and error values do not always match the
  real `os` backend. Tests that pass against MemMapFs can still fail against
  OsFs (and vice versa) on edge cases like path separators on Windows, `Chmod`
  semantics, or `os.IsNotExist` error identity. Treat MemMapFs as a fast
  approximation, not a conformance oracle; keep at least a thin integration test
  on OsFs.
- **In-memory means in-RAM.** MemMapFs holds file contents in memory with no
  size ceiling. Streaming large files or many fixtures through it in tests can
  balloon memory; it is not a spill-to-disk cache.
- **Cloud/network backends are second-class.** GCS and SFTP ship in-repo but are
  marked experimental; S3, MinIO, Dropbox, and Google Drive live in third-party
  modules (notably the `fclairamb/*` and `unmango/aferox` families). These vary
  in maintenance and often cannot honor the full `Fs`/`File` contract — object
  stores have no real directories, no atomic rename, and frequently no
  write-seek. Code written against `OsFs` will not necessarily "just work" on an
  S3-backed `Fs`; the abstraction leaks exactly where object storage differs
  from POSIX.
- **Interface, not zero-cost.** Every filesystem call goes through an interface
  dispatch and, for composed filesystems, one wrapper per layer. This is
  negligible for config files but measurable in tight loops over many small
  files; keep hot paths shallow.
- **Prefer `io/fs` for read-only.** If a function only reads, take
  `fs.FS`/`fs.ReadFileFS` from the standard library rather than `afero.Fs`.
  Reserve Afero for code that writes, mutates, or composes.
- **Requires a recent Go.** Current `master` targets Go >= 1.23[^4]. Pin an
  older Afero tag if you must build on an older toolchain.

## When to Use / When Not

**Use when:**
- You want filesystem-dependent code to be unit-testable in memory with no disk
  fixtures or cleanup.
- Your code needs to **write/modify/delete**, not just read, and you still want
  a swappable backend.
- You genuinely benefit from composition: sandboxing writes (CopyOnWrite),
  caching a slow store, or jailing access to a subtree (BasePathFs).
- You are already in the Hugo/Cobra/Viper ecosystem, where Afero is idiomatic.

**Avoid when:**
- You only read files — use the standard library `io/fs` and `embed`.
- You want a cloud object-store abstraction with correct semantics — use a
  blob-oriented library instead of forcing S3 into a POSIX file interface.
- You are on a hot path over huge trees where interface dispatch and MemMapFs
  memory use matter.

## Alternatives

- Go stdlib `io/fs` (golang/go) — use instead when your code is read-only;
  it is the standard, dependency-free interface and pairs with `embed`.
- hack-pad/hackpadfs — use when you want a writable filesystem interface built
  natively around `io/fs` conventions rather than mirroring `os`.
- go-git/go-billy — use when you need a filesystem abstraction that integrates
  with go-git; it is that ecosystem's equivalent of Afero.
- google/go-cloud (`gocloud.dev/blob`) — use instead when you actually want
  cloud object storage (S3/GCS/Azure) with a blob model, not a file model.
- C2FO/vfs — use when you want a unified abstraction spanning local and cloud
  (S3/GCS/SFTP) designed for that mix from the start.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-10 | First commit; extracted from spf13's Hugo/Viper work[^1]. |
| v1.x | 2016+ | Settled on the `afero.Fs`/`afero.File` interfaces and `MemMapFs`; adopted across Hugo, Cobra, Viper[^2]. |
| io/fs bridge | 2021 | `NewIOFS`/`FromIOFS` added to interoperate with Go 1.16 `io/fs` and `embed`[^3]. |
| current | 2026-07 | Actively maintained (last push 2026-07-08); Go >= 1.23; in-repo GCS/SFTP still experimental[^4]. |

Afero uses fine-grained patch releases rather than large numbered milestones, so
the meaningful history is capability-based (interface stabilization, `io/fs`
interop) rather than a headline version cadence.

## References

[^1]: Afero README — "The Universal Filesystem Abstraction for Go." https://github.com/spf13/afero
[^2]: Steve Francia (spf13) is the author of Hugo, Cobra, and Viper, all of which consume Afero for filesystem access. https://github.com/spf13
[^3]: Go 1.16 release notes — introduction of the `io/fs` package (2021). https://go.dev/doc/go1.16#fs
[^4]: Repository metadata via GitHub API (2026-07): Apache-2.0, default branch `master`, last push 2026-07-08, Go version badge `>=1.23`. https://github.com/spf13/afero

## Tags

go, golang, filesystem, virtual-filesystem, testing, in-memory, io-fs, abstraction, storage, composition, spf13
