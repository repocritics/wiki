# spatie/laravel-medialibrary

> Associate files with Eloquent models, with on-the-fly image/PDF/video conversions handled through Laravel's Filesystem.

[GitHub repo](https://github.com/spatie/laravel-medialibrary) ·
[Official website](https://spatie.be/docs/laravel-medialibrary) ·
[License: MIT](https://github.com/spatie/laravel-medialibrary/blob/main/LICENSE.md)

## Overview

Laravel Medialibrary is Spatie's package for attaching arbitrary files to Eloquent
models. A model implements the `HasMedia` interface and uses the
`InteractsWithMedia` trait; from then on you call `$model->addMedia($file)->toMediaCollection('images')`
and the package records a row in a polymorphic `media` table and copies the file to
a configured Laravel disk[^1]. Every attached file becomes a `Media` Eloquent model,
so ordinary relationship, query, and event mechanics apply.

Its second job is derivative generation. On top of storing the original, it can
produce "media conversions" — resized/reformatted images, PDF thumbnails, video
frame grabs — declared per-model in `registerMediaConversions()` and rendered via
Spatie's own `spatie/image` layer over GD or Imagick[^2]. Conversions and responsive
image variants are, by default, generated on a queue, which is the single most
important operational fact about the package: without a running queue worker,
derivatives silently never appear.

The defining tension is convenience versus a large surface area. For CRUD apps that
need "upload an avatar, show a thumbnail," it removes a lot of boilerplate. But it
owns a database table, a filesystem layout, a queue dependency, and an image toolchain
(GD/Imagick, plus Ghostscript for PDFs and FFmpeg for video). The upload UI itself is
not in the free package — polished Blade/Livewire/Vue/React upload components live in
the separate paid **Media Library Pro** product[^3].

## Getting Started

```bash
composer require spatie/laravel-medialibrary
php artisan vendor:publish --tag="medialibrary-migrations"
php artisan migrate
```

```php
use Illuminate\Database\Eloquent\Model;
use Spatie\MediaLibrary\HasMedia;
use Spatie\MediaLibrary\InteractsWithMedia;
use Spatie\MediaLibrary\MediaCollections\Models\Media;

class News extends Model implements HasMedia
{
    use InteractsWithMedia;

    public function registerMediaConversions(?Media $media = null): void
    {
        $this->addMediaConversion('thumb')
            ->width(300)->height(300)
            ->nonQueued();          // generate inline instead of on the queue
    }
}

// Attach an uploaded file and read back a URL
$news = News::find(1);
$news->addMedia($request->file('image'))->toMediaCollection('images');
$url  = $news->getFirstMediaUrl('images', 'thumb');
```

## Architecture / How It Works

Each attached file is one row in the `media` table with a polymorphic
`model_type`/`model_id` pair plus `uuid`, `collection_name`, `disk`, `file_name`,
`size`, `mime_type`, and JSON `custom_properties`, `manipulations`, and
`generated_conversions` columns[^1]. The physical layout is decided by a swappable
`PathGenerator`; the default stores each media item in a directory named after its
primary key (`{id}/file.jpg`, conversions under `{id}/conversions/`). URLs come from
a parallel `UrlGenerator`. Both are interfaces you can replace, which is how people
adapt the package to CDNs or pre-existing storage schemes.

Storage is delegated entirely to Laravel's Filesystem, so any configured disk works —
`local`, `public`, `s3`, etc. — and a media item can live on a different disk than its
model's other files. When a file is added from a remote disk, the package pulls it to
a local temp directory to run conversions, then pushes results back.

Conversions are the moving part. `registerMediaConversions()` builds a chain of
`spatie/image` manipulations (fit, format, quality, optimize). By default the actual
work is dispatched as a `PerformConversionsJob` onto the queue; `->nonQueued()` forces
inline generation. `->withResponsiveImages()` additionally slices an image into many
widths plus a base64 tiny placeholder for `srcset` output. PDF thumbnails require
Ghostscript; video frame extraction requires FFmpeg. Deleting a model deletes its media
rows and files through model events, and deleting a `Media` row cleans up its files —
but bypassing Eloquent (raw SQL, truncate) leaves orphaned files behind.

## Production Notes

- **The queue is not optional in practice.** With the default queued conversions,
  a missing or crashed worker means thumbnails never generate and no error surfaces in
  the request path. Either run a reliable worker or mark conversions `->nonQueued()`
  and accept slower uploads.
- **Image conversion is memory-hungry.** Imagick/GD load full bitmaps into memory;
  large source images (high-megapixel phone photos, scanned PDFs) can blow past PHP's
  `memory_limit` and the queue worker's limits. Cap upload dimensions or raise worker
  memory deliberately.
- **S3 round-trips.** On remote disks every conversion is download-to-temp, process,
  re-upload; high-volume ingestion is I/O bound and benefits from a local staging disk.
- **Regeneration.** Changing a conversion definition does not retroactively update
  existing media — run `php artisan media-library:regenerate` (scopable by model/ids),
  a long queue-heavy batch on large libraries.
- **N+1.** Reading media in a loop hits the DB per model; eager-load the `media`
  relation (`->with('media')`).
- **Cleanup.** Orphaned files and dangling conversions are swept by
  `media-library:clean`; schedule it rather than assuming deletes always cascade.
- **Upgrades cross major versions rename the core trait/interface** (older code used
  `HasMediaTrait`; current is `InteractsWithMedia` + the `HasMedia` interface) and have
  shifted PHP/Laravel floors. Read `UPGRADING.md` before bumping majors[^4].

## When to Use / When Not

**Use when:**
- You want file attachments tied to models with minimal glue and a disk-agnostic
  storage story.
- You need generated thumbnails/derivatives and already run (or will run) a queue.
- You want per-collection rules (single file, MIME whitelists, fallback URLs).

**Avoid when:**
- You only need to manipulate an image once and hand back a path — reach for
  `intervention/image` or `spatie/image` directly without the table and lifecycle.
- You cannot run a queue worker and inline conversion latency is unacceptable.
- Your files are not naturally owned by a single model (shared asset library,
  document management with its own taxonomy) — the polymorphic ownership model fits
  poorly.

## Alternatives

- plank/laravel-mediable — similar Eloquent file association with a many-to-many
  attach model; use it when you want files shared across multiple models rather than
  owned by one.
- intervention/image — pure image manipulation; use it when you need conversions but
  not the database/collection machinery.
- spatie/image — the manipulation layer Medialibrary sits on; use directly for
  one-off processing without persistence.
- CodeSleeve/laravel-stapler — older Paperclip-style attachment package; largely
  legacy, choose only for maintaining existing installs.
- talvbansal/media-manager — a file-browser/manager UI rather than a model-attachment
  layer; use when you want an admin media browser, not per-model attachments.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015 | Initial release; associate files with Eloquent models[^1]. |
| 7.0 | 2019 | Major rewrite: `HasMedia`/`InteractsWithMedia` API, responsive images, conversions overhaul[^4]. |
| 8.0 | 2020 | Laravel 8 support; dropped older PHP/Laravel floors. |
| 9.0 | 2021 | Laravel 9 support. |
| 10.0 | 2023 | Laravel 10 support, PHP 8.1+ floor. |
| 11.0 | 2024 | Laravel 11 support, PHP 8.2+ floor; current major line (docs v11)[^1]. |

## References

[^1]: Laravel Medialibrary documentation (v11). https://spatie.be/docs/laravel-medialibrary
[^2]: spatie/image — image manipulation library used for conversions. https://github.com/spatie/image
[^3]: Media Library Pro — paid upload UI components (Blade/Livewire/Vue/React). https://medialibrary.pro
[^4]: UPGRADING.md — cross-major upgrade notes. https://github.com/spatie/laravel-medialibrary/blob/main/UPGRADING.md

## Tags

php, laravel, eloquent, file-upload, media-management, image-processing, orm, storage, spatie, conversions
