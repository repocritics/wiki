# paperless-ngx/paperless-ngx

> A self-hosted document management system that OCRs, indexes, and archives your scanned paper into a searchable database.

[GitHub repo](https://github.com/paperless-ngx/paperless-ngx) ·
[Official website](https://docs.paperless-ngx.com/) ·
[License: GPL-3.0](https://github.com/paperless-ngx/paperless-ngx/blob/main/LICENSE)

## Overview

Paperless-ngx turns a pile of scanned PDFs and images into a full-text-searchable
archive with automatic tagging, correspondent detection, and document-type
classification. You point a scanner or a watched folder at it, and it runs OCR,
extracts text, generates a PDF/A archive copy, and files the document under
metadata it either infers or you assign. It is one of the canonical self-hosted
"replace the filing cabinet" tools in the homelab space.

The project is the third generation of a lineage: the original **Paperless** by
Daniel Quinn, then **Paperless-ng** by Jonas Winkler, then **Paperless-ngx**,
which forked in 2022 when paperless-ng stalled and a group of contributors chose
to distribute maintenance across a team rather than depend on a single
author[^1]. That governance choice is the defining feature — "ngx" exists
precisely so the project does not die when one maintainer walks away, and it has
kept a steady release cadence since.

The defining tension is **security posture versus convenience**. Paperless stores
your documents and their extracted text in clear, unencrypted form on disk and in
the database. The maintainers are explicit that it should never run on an
untrusted host[^2]. It is aimed at people who will run it on a home server or a
private VPS they control — not a multi-tenant SaaS, not a shared box.

## Getting Started

The supported install path is Docker Compose via the bootstrap script:

```bash
bash -c "$(curl -L https://raw.githubusercontent.com/paperless-ngx/paperless-ngx/main/install-paperless-ngx.sh)"
```

This scaffolds a `docker-compose.yml` plus a `docker-compose.env`, wiring together
the webserver, a database (PostgreSQL by default), Redis, and — if you opt in —
Gotenberg and Apache Tika for Office-document conversion. Once running, drop a
file into the consumption directory and it appears in the web UI after OCR:

```bash
# consume a document by copying it into the watched folder
cp ~/scans/invoice-2026-07.pdf ./consume/

# tail the consumer to watch OCR + classification happen
docker compose logs -f webserver
```

The web UI (default `http://localhost:8000`) is where you create tags,
correspondents, document types, storage paths, and — since v2 — custom fields and
workflows.

## Architecture / How It Works

Paperless-ngx is a Django (Python) backend with an Angular single-page frontend,
served through gunicorn/uvicorn with Django Channels providing the websocket that
streams live consumption status to the browser.

The pipeline has distinct stages, coordinated by a **Celery** task queue backed by
**Redis** (the project migrated off django-q to Celery early in the ngx line):

1. **Consumer** — watches the consumption directory (or receives uploads / mail /
   API posts), deduplicates by checksum, and enqueues a consume task.
2. **OCR** — runs **OCRmyPDF**, which wraps **Tesseract**, to extract a text layer
   and produce a PDF/A archive version alongside the original[^3]. Office formats
   are routed through **Gotenberg** and **Apache Tika** first.
3. **Classification** — a **scikit-learn** model auto-assigns tags,
   correspondents, document types, and storage paths. Matching can be automatic
   (the trained classifier) or rule-based (literal / regex / fuzzy) per object.
4. **Index** — extracted text is written to a **Whoosh** full-text index for
   search; metadata lands in the relational database.

Storage is split: the **original** file is kept untouched, and an **archive**
PDF/A version is generated for viewing and search. Both live on the filesystem;
only metadata and the search index are in the database and Redis. This means the
document files are portable and can be re-indexed, but also means a backup must
capture the filesystem and the database as a consistent pair.

**Workflows** (v2.0) generalized the older consumption-template system into
triggers and actions — assign metadata, run at consumption or on update, fire
webhooks. **Custom fields** (also v2.0) added typed metadata beyond the built-in
tag/correspondent/type triad.

## Production Notes

**OCR is the bottleneck, and it is CPU-bound.** Consumption throughput is
dominated by Tesseract. On a low-power NAS, large or multi-page scans can take
minutes each. Tune `PAPERLESS_TASK_WORKERS` and `PAPERLESS_THREADS_PER_WORKER`
to your core count; over-provisioning workers on a small box causes memory
pressure rather than speedup. `PAPERLESS_OCR_MODE=skip` avoids re-OCRing files
that already carry a text layer.

**Postgres over SQLite for anything real.** SQLite works and is the zero-config
default in some setups, but for libraries beyond a few thousand documents,
PostgreSQL is the recommended and better-tested path. MariaDB is supported but is
the least-exercised of the three.

**Backups are a filesystem + database pairing.** The safe backup primitive is the
built-in `document exporter` management command, which serializes documents and
metadata into a portable directory you can re-import. Backing up the raw data
directory plus a DB dump also works but must be point-in-time consistent — a DB
snapshot that disagrees with the file store leaves orphaned or missing documents.
The archive versions and search index are regenerable, so they are not strictly
required in a backup.

**The Whoosh index can drift or corrupt.** If search returns stale or wrong
results, `document index reindex` rebuilds it from the database. Similarly
`document create_classifier` retrains the auto-matching model.

**No encryption at rest.** GPG document encryption existed in older Paperless and
was removed; there is no built-in at-rest encryption today. If you need it, put
the data directory on an encrypted volume (LUKS, etc.) at the OS level. This is
the direct consequence of the "never run on an untrusted host" warning.

**Upgrades are usually drop-in but read the notes.** Pulling a new image and
letting Django migrations run covers most upgrades. Major version bumps (notably
1.x → 2.0) occasionally change defaults or require a reindex; the release notes
call these out, and skipping several majors at once is riskier than incremental
steps.

**Reverse proxy and `PAPERLESS_URL`.** Running behind nginx/Traefik with HTTPS is
standard, but you must set `PAPERLESS_URL` (and CSRF/allowed-host vars) correctly
or logins and websocket status updates silently fail.

## When to Use / When Not

**Use when:**
- You want a self-hosted, searchable archive of scanned paper on hardware you
  control.
- You value automatic OCR + classification and are willing to run a small stack
  (Django + Redis + Postgres + OCR).
- You want an actively maintained, team-governed project rather than a
  single-maintainer tool.

**Avoid when:**
- You need a multi-tenant or public-internet-exposed SaaS — the clear-text
  storage model is a poor fit.
- You want built-in end-to-end or at-rest encryption without OS-level volume
  encryption.
- You need collaborative editing, versioned office documents, or a records-
  management/compliance workflow (retention policies, legal holds) — this is an
  archive, not an ECM suite.
- Your hardware is too weak to OCR at an acceptable rate and you cannot pre-OCR
  upstream.

## Alternatives

- docspell/docspell — JVM-based DMS with strong auto-tagging; use instead when you
  prefer a Scala stack and organization-oriented features.
- mayan-edms/mayan-edms — heavier, more enterprise-ECM DMS with workflows and
  permissions; use when you need finer access control at the cost of complexity.
- papermerge/papermerge-core — another Python/Django OCR DMS with a page-oriented
  model; use when you want document/page reordering as a first-class concept.
- sismics/docs (Teedy) — Java DMS with a clean UI; use when you want simpler
  sharing and less of an OCR-pipeline focus.
- the-paperless-project/paperless — the original, now unmaintained; only relevant
  as historical lineage, not for new deployments.

## History

| Version | Date | Notes |
|---------|------|-------|
| paperless | ~2015 | Original project by Daniel Quinn. |
| paperless-ng | ~2020 | Jonas Winkler's rewrite; new Angular UI, ML matching[^1]. |
| ngx fork | 2022-02 | paperless-ngx org created; team-maintained successor[^1]. |
| 1.x | 2022 | Celery task queue, consumption pipeline maturation. |
| 2.0 | 2023-11 | Custom fields, workflows engine, frontend overhaul[^4]. |

## References

[^1]: Paperless-ngx README — lineage from Paperless and Paperless-ng, and the team-maintenance rationale. https://github.com/paperless-ngx/paperless-ngx
[^2]: Paperless-ngx README, "Important Note" — clear-text storage; never run on an untrusted host. https://github.com/paperless-ngx/paperless-ngx#important-note
[^3]: OCRmyPDF — the OCR engine wrapper (Tesseract) used to add text layers and produce PDF/A. https://github.com/ocrmypdf/OCRmyPDF
[^4]: Paperless-ngx documentation and release notes. https://docs.paperless-ngx.com/

## Tags

python, django, angular, document-management, ocr, self-hosted, dms, pdf, full-text-search, archiving, homelab
