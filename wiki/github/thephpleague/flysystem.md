# thephpleague/flysystem

> One PHP interface over local disks, FTP/SFTP, and object stores like S3 — the storage layer underneath Laravel's `Storage` facade.

[GitHub repo](https://github.com/thephpleague/flysystem) ·
[Official website](https://flysystem.thephpleague.com) ·
[License: MIT](https://github.com/thephpleague/flysystem/blob/3.x/LICENSE)

## Overview

Flysystem is a filesystem abstraction library for PHP by Frank de Jonge, maintained under The PHP League[^1]. It exposes a single `Filesystem` object whose methods — `write`, `read`, `delete`, `listContents`, `move`, `copy`, visibility and metadata calls — behave the same whether the bytes land on a local disk, an FTP server, or an S3 bucket. The concrete backend is a swappable `FilesystemAdapter`. Its reach far exceeds its star count: Laravel's `Storage` facade is built on Flysystem, so most of the Laravel ecosystem depends on it transitively, and `league/flysystem` sits among the most-downloaded Packagist packages of all time[^2].

The library's defining event is the 2.0 rewrite (2021). V1 was forgiving and feature-broad: methods returned `false` on failure, adapters were bundled in the core package, and a plugin/caching system let you decorate the filesystem[^3]. V2 threw all of that out. Failures now raise typed exceptions (`UnableToWriteFile`, `UnableToReadFile`, …), adapters were split into separate Composer packages, the plugin and metadata-cache systems were removed, and the public surface was tightened and made strict[^4]. V3 followed within the same year, keeping the V2 contract and adding features like unified checksums. The result is a cleaner, more predictable library — and a hard upgrade wall for anyone still on 1.x.

The central tension is honesty versus uniformity. Flysystem promises one API across wildly different backends, but the abstraction is deliberately lossy: it models only what all backends can agree on. Visibility collapses to `public`/`private`, there are no true directories on S3, and semantics like atomic moves or last-modified precision differ underneath a call that looks identical everywhere.

## Getting Started

Adapters are separate packages as of V2/V3 — install the core plus the one you need:

```bash
composer require league/flysystem league/flysystem-aws-s3-v3
```

```php
<?php
use League\Flysystem\Filesystem;
use League\Flysystem\AwsS3V3\AwsS3V3Adapter;
use Aws\S3\S3Client;

$client = new S3Client([
    'region'  => 'us-east-1',
    'version' => 'latest',
]);

$adapter = new AwsS3V3Adapter($client, 'my-bucket', 'optional/prefix');
$filesystem = new Filesystem($adapter);

// Strict API: these throw FilesystemException on failure, not return false.
$filesystem->write('reports/2026.txt', "hello\n");
$contents = $filesystem->read('reports/2026.txt');

// listContents returns a lazy iterable — you must iterate it.
foreach ($filesystem->listContents('reports', deep: true) as $item) {
    echo $item->path(), PHP_EOL;
}
```

For a local disk, swap in `League\Flysystem\Local\LocalFilesystemAdapter` from `league/flysystem-local`.

## Architecture / How It Works

`Filesystem` (implementing `FilesystemOperator`) is a thin, backend-agnostic facade. It normalizes and prefixes paths, then delegates every operation to a `FilesystemAdapter`. The adapter interface is narrow by design — roughly two dozen methods covering existence, read/write (string and stream variants), delete, directory create/delete, listing, move/copy, visibility, and metadata (`fileSize`, `mimeType`, `lastModified`). Everything Flysystem can do is exactly what that interface exposes; there is no escape hatch to backend-specific features without dropping to the underlying client.

Key internals:

- **Path handling** — a `PathNormalizer` (default `WhitespacePathNormalizer`) rejects path traversal and normalizes separators; a `PathPrefixer` transparently scopes every operation under a configured root prefix. Leading slashes are stripped; paths are always relative to the adapter root.
- **Streams first** — `writeStream`/`readStream` take and return PHP resources so large files never have to be fully buffered in memory. The string variants are convenience wrappers.
- **Visibility** — an abstract `public`/`private` model. On local, a `PortableVisibilityConverter` maps these to Unix permission bits; on S3 it maps to canned ACLs. The mapping is configurable but the vocabulary is fixed at two values.
- **Listing** — `listContents()` returns a lazy `DirectoryListing` (a generator wrapper). Nothing is fetched until you iterate, and `deep: true` vs shallow changes whether the adapter recurses.
- **MountManager** — an optional multi-filesystem coordinator that lets you address filesystems by scheme (`local://…`, `s3://…`) and copy/move across them.

The core package ships no adapters. Official adapters (`-local`, `-ftp`, `-sftp-v3`, `-aws-s3-v3`, `-async-aws-s3`, `-google-cloud-storage`, `-in-memory`, `-webdav`, `-ziparchive`, `-gridfs`) and a large third-party set are each versioned independently, so you upgrade the core and your adapters on separate cadences.

## Production Notes

**The V1 → V2/V3 upgrade is a rewrite migration, not a version bump.** Return-value checks (`if ($fs->write(...) === false)`) must become try/catch around `FilesystemException`. `has()` split into `fileExists()` and `directoryExists()`. The plugin system and cached adapters are gone with no drop-in replacement — metadata caching now has to be built as a decorating adapter or handled by a third-party package[^4].

**S3 has no directories.** `directoryExists()`, `createDirectory()`, and per-object metadata calls each cost API round-trips, and prefixes are simulated. Treating S3 like a POSIX filesystem — probing existence in tight loops, relying on directory semantics — quietly turns into request-count and latency problems. Batch and cache at the application layer.

**Metadata calls are not free and not always reliable.** `mimeType()` may be derived from `finfo` on content or from the extension depending on adapter and configuration; it can disagree with reality. `fileSize`/`lastModified` on remote adapters are individual requests. Don't call them speculatively inside listing loops when `listContents` can already carry the metadata.

**Moves and atomicity vary.** `move()` is a native rename on local, but copy-then-delete on backends without atomic rename — non-atomic and non-cheap for large objects on S3. There is no cross-adapter transaction; a failed move can leave a partial copy.

**Laravel pins the major version.** Because Laravel's `Storage` is built on Flysystem, the Flysystem major is constrained by your Laravel version (Laravel 9+ ships Flysystem 3). You generally cannot bump Flysystem independently of the framework without dependency conflicts, so plan Flysystem upgrades alongside framework upgrades.

**Visibility is lossy.** Only `public`/`private` cross the abstraction. Fine-grained ACLs, ownership, or SELinux contexts are outside the model — set them on the backend directly if you need them.

## When to Use / When Not

**Use when:**
- You want to swap local disk for S3/GCS/FTP without rewriting call sites.
- You're in Laravel and using `Storage` — you're already using it.
- You want strict, exception-based error handling and streaming for large files.
- You need to write filesystem-agnostic library code that others plug backends into.

**Avoid when:**
- You need backend-specific features (S3 presigned URLs beyond basics, multipart control, object tagging, ACL nuance) — use the vendor SDK directly, or alongside it.
- You need true atomic cross-store operations or transactional guarantees.
- You're only ever touching the local disk and don't want an abstraction layer — plain `fopen`/`file_put_contents` is lighter.
- You're on PHP 7.x — V2/V3 require PHP 8.0.2+; only the unmaintained 1.x line supports older PHP.

## Alternatives

- aws/aws-sdk-php — use directly when you need full S3 semantics (presigned URLs, tagging, multipart) rather than a lowest-common-denominator abstraction.
- symfony/filesystem — use for local filesystem operations and path utilities when you have no remote backend to abstract over.
- gaufrette/gaufrette — the older PHP filesystem-abstraction library; use only if maintaining a legacy codebase already built on it.
- spatie/flysystem-dropbox — use when you specifically need Dropbox as a Flysystem backend (community adapter on top of this library).
- Native `fopen`/stream wrappers — use for simple single-backend scripts where an abstraction adds no value.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x–1.x | 2013–2014 | Original release. Bundled adapters, `false`-on-failure returns, plugin and metadata-cache systems[^3]. |
| 2.0 | 2021 | Full rewrite. Typed exceptions, adapters split into separate packages, plugins/caching removed, PHP 8+ required[^4]. |
| 3.0 | 2021 | Builds on the V2 contract; unified checksum support and continued adapter split. Current major; default branch `3.x`. |

## References

[^1]: Flysystem documentation and source, The PHP League. https://flysystem.thephpleague.com and https://github.com/thephpleague/flysystem
[^2]: `league/flysystem` on Packagist (download statistics, adapter packages). https://packagist.org/packages/league/flysystem
[^3]: Flysystem 1.x documentation (bundled adapters, plugins, cached adapters) — archived in the project docs and 1.x branch history.
[^4]: Flysystem "What is new in V2/V3" and "Upgrade from 1.x" guides. https://flysystem.thephpleague.com/docs/what-is-new/ and https://flysystem.thephpleague.com/docs/upgrade-from-1.x/

## Tags

php, filesystem, storage-abstraction, s3, object-storage, laravel, ftp, sftp, cloud-storage, php-library, adapter-pattern
