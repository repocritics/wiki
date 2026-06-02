# Stirling-Tools/Stirling-PDF

Stirling-PDF — a locally-hosted, web-based PDF manipulation tool. Merge, split, rotate, OCR, convert, sign, edit.

## What it is

A Java + Spring Boot web app that provides ~50 PDF operations (merge, split, rotate, watermark, OCR, sign, encrypt, convert to/from Word/HTML/images, more) via a browser UI. Self-hosted via Docker — no PDFs leave your machine. Aimed at users who don't want to upload sensitive documents to commercial PDF SaaS tools.

## Key features

- 50+ PDF operations across merge/split/rotate/OCR/convert/sign/edit categories.
- Web UI accessible from any device on the local network.
- Self-host via Docker.
- OCR support via Tesseract.
- API for headless usage.
- MIT-licensed.

## Tech stack

- Java + Spring Boot on the backend.
- TypeScript at the language tag — companion JS for the UI.
- Distributed via Docker.

## When to reach for it

- You want a self-hosted PDF toolkit for sensitive documents.
- You're tired of uploading PDFs to commercial SaaS for simple edits.
- You're standing up internal tools for a privacy-sensitive org.

## When *not* to reach for it

- You want a desktop GUI — Stirling-PDF is a web service.
- You need professional-grade PDF editing — Adobe Acrobat, Foxit are commercial options.

## Maturity signal

Actively maintained.

## Alternatives

- SmallPDF, ILovePDF — commercial SaaS.
- pdftk + custom CLI — for CLI-only workflows.
- LibreOffice Draw — desktop GUI PDF editing.

## Tags

java, spring-boot, pdf, self-hosted, mit-license, ocr, document-processing
