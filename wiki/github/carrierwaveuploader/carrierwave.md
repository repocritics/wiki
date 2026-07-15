# carrierwaveuploader/carrierwave

> File-upload library for Rails and Rack apps that models each upload as an explicit Ruby "uploader" class.

[GitHub repo](https://github.com/carrierwaveuploader/carrierwave) ·
[Documentation](https://rubydoc.info/gems/carrierwave) ·
[License: MIT](https://github.com/carrierwaveuploader/carrierwave/blob/master/carrierwave.gemspec)

## Overview

CarrierWave is a Ruby gem for handling file uploads, first released around 2009 by
Jonas Nicklas and maintained since under the `carrierwaveuploader` organization[^1].
It predates Rails' built-in ActiveStorage by nearly a decade and, along with
Paperclip, defined how a generation of Rails apps handled avatars, attachments,
and image thumbnails. It works with any Rack-based framework (Rails, Sinatra) and
integrates with ActiveRecord out of the box; Mongoid, Sequel, and DataMapper support
live in separate gems.

The defining idea is the **uploader class**: instead of configuring uploads through
model options, you subclass `CarrierWave::Uploader::Base` and express storage
location, allowed types, processing, and derived versions as Ruby methods and a
small DSL. That uploader is then *mounted* onto a model column with `mount_uploader`.
This gives fine-grained, per-attachment control at the cost of more moving parts than
convention-driven alternatives.

The central tension is that a mounted column stores only the **filename identifier**,
not a path or URL. The public URL is reconstructed at read time from `store_dir` plus
that identifier — which keeps the database small but couples every stored file to the
uploader's `store_dir`/`filename` logic. Change either after files exist and the old
files become unreachable; much of operating CarrierWave is about not breaking that
reconstruction contract. As of 2026 the project is mature and still maintained (last
release 3.1.3, May 2026) but no longer the default for new Rails apps, which reach for
ActiveStorage or Shrine.

## Getting Started

```ruby
# Gemfile
gem 'carrierwave', '~> 3.0'
```

```bash
rails generate uploader Avatar   # => app/uploaders/avatar_uploader.rb
rails g migration add_avatar_to_users avatar:string && rails db:migrate
```

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  include CarrierWave::MiniMagick
  storage :file
  process resize_to_fit: [800, 800]

  version :thumb do
    process resize_to_fill: [200, 200]
  end
end

class User < ApplicationRecord
  mount_uploader :avatar, AvatarUploader
end

user.avatar = params[:file]   # cached
user.save!                    # promoted to permanent store
user.avatar.url               # => "/uploads/.../photo.png"
user.avatar.thumb.url         # => ".../thumb_photo.png"
```

## Architecture / How It Works

**Two-phase cache/store lifecycle.** An assigned file first lands in a temporary
**cache** (`cache_dir`, a timestamped directory) and is only promoted to permanent
**store** when the model is saved. This is what lets an upload survive a failed
validation and a form redisplay: a hidden `avatar_cache` field carries the cache id
across the round-trip so the user does not re-pick the file. The store backend is
pluggable — `storage :file` for the local filesystem, `storage :fog` for S3, Google,
and other clouds via the `fog` gems.

**Uploader as a mixin stack.** `CarrierWave::Uploader::Base` is assembled from modules
(Cache, Store, Processing, Versions, Download, Configuration). Processing engines are
opt-in mixins: `CarrierWave::MiniMagick` (shelling out to ImageMagick) or the older
RMagick binding. The `process` DSL registers callbacks; the `version` DSL declares
derived files (thumbnails) that are generated after the base processing runs.

**Versions** are independent processed derivatives stored alongside the original with a
prefixed filename (`thumb_photo.png`). Versions can nest, be conditional (`if:`), and be
derived from earlier versions (`from_version:`) to avoid re-processing the full-size
image. Because versions reconstruct their own filenames on retrieval, customizing a
version filename requires overriding `full_filename` rather than `filename`.

**ORM coupling.** `mount_uploader` hooks the upload into the ActiveRecord save
lifecycle — assignment caches, `save` stores, `destroy` removes. Processing runs
**synchronously inside that callback**, so image manipulation happens in the web request
by default. `mount_uploaders` (plural) handles multiple files by serializing an array of
identifiers into a JSON/array column.

## Production Notes

**Synchronous processing blocks requests.** Every version is generated in-process during
`save`. A handful of thumbnails on a large image can add seconds to a request. There is no
built-in background story; the historical answer, `carrierwave_backgrounder`, is
effectively unmaintained. The practical pattern is to store the original, then call
`recreate_versions!` from a background job.

**Cached files leak disk.** Files written to `cache_dir` are not cleaned automatically.
Over time an abandoned-upload directory grows unbounded. You must schedule
`CarrierWave.clean_cached_files!` (e.g. a daily cron/rake task) or the cache accumulates
forever — a common cause of "disk full" on long-running boxes.

**`store_dir` is a contract, not a setting.** Because the DB stores only the identifier,
`store_dir` must be deterministic and stable for the life of the data. Two classic
footguns: (1) using `model.id` in `store_dir` before the record is persisted (id is `nil`
on a new record, so cached and stored paths diverge); (2) refactoring `store_dir` later,
which silently orphans every previously stored file.

**Security: CVE-2016-3714 (ImageTragick).** CarrierWave can mitigate it, but only a
`content_type_allowlist` (which inspects file headers) is effective — `extension_allowlist`
checks the name only and leaves the app exposed. You must also disable or replace
ImageMagick's SVG delegate. Do not rely on extension checks for untrusted uploads.

**Mounted accessor never returns nil.** `user.avatar` is always a truthy uploader object;
test presence with `user.avatar.file.nil?` or `user.avatar?`, not `if user.avatar`.

**Upgrade pains.** 3.0 changed file-extension handling on `process convert:` — set
`force_extension false` to keep 2.x behavior, and note that 2.x and 3.x produce
incompatible cache filenames, which breaks blue-green deploys that serve both at once[^2].
`recreate_versions!` does not persist the new filename when random filenames are used, so
you must call `save!` yourself afterward to keep DB and disk in sync.

**Cloud storage weight.** `fog`-based backends pull in large dependency trees; many teams
use the lighter `carrierwave-aws` adapter for S3-only setups. Uploads always transit the
app server — CarrierWave has no built-in direct-to-S3 browser upload.

## When to Use / When Not

**Use when:**
- You're maintaining an existing Rails app already built on CarrierWave.
- You want explicit per-attachment uploader classes with fine control over storage dirs,
  filenames, and image versions.
- You need synchronous thumbnail generation with a small, well-understood API.

**Avoid when:**
- Starting a new Rails app — ActiveStorage ships with Rails and needs no gem.
- You need direct-to-S3 uploads, first-class background processing, or a plugin system —
  Shrine is designed for that.
- Your workload processes large or many images per request and can't tolerate synchronous
  in-request processing without extra engineering.

## Alternatives

- rails/rails (ActiveStorage) — built into Rails, direct uploads, zero extra gem; use it for new apps that don't need per-uploader customization.
- shrine/shrine — modular plugin architecture, background jobs, direct S3 uploads; use when you need flexibility and performance beyond CarrierWave's model.
- thoughtbot/paperclip — the other classic Rails uploader, now deprecated (EOL, migrate to ActiveStorage); only relevant when porting legacy code off it.
- refile/refile — direct-upload-focused but unmaintained; avoid for new work.

## History

| Version | Date | Notes |
|---------|------|-------|
| ~0.1 | 2009 | Initial release by Jonas Nicklas[^1]. |
| 1.0.0 | 2016-12-24 | First 1.x; `mime-types` runtime dep, content-type set automatically. |
| 2.0.0 | 2021-02-23 | Major release; dropped older Ruby/Rails support[^3]. |
| 3.0.0 | 2023-07-02 | File-extension-on-convert change; `force_extension` for 2.x behavior[^2]. |
| 3.1.3 | 2026-05-23 | Latest 3.x maintenance release. |

## References

[^1]: CarrierWave gem — RubyDoc and gem metadata. https://rubydoc.info/gems/carrierwave
[^2]: "Upgrading from 2.x or earlier" and PR #2659 — file extension handling on conversion. https://github.com/carrierwaveuploader/carrierwave/pull/2659
[^3]: CarrierWave CHANGELOG. https://github.com/carrierwaveuploader/carrierwave/blob/master/CHANGELOG.md

## Tags

ruby, rails, file-upload, rack, activerecord, image-processing, cloud-storage, s3, thumbnails, gem
