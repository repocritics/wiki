# laurent22/joplin

Joplin — a privacy-focused, open-source note-taking app with sync across Windows, macOS, Linux, Android, iOS. Markdown-first; encrypted sync via your chosen backend.

## What it is

A note + to-do app with end-to-end encrypted sync. Notes live in Markdown; sync targets are user-chosen (Joplin Cloud, Dropbox, OneDrive, Nextcloud, WebDAV, or self-hosted). Desktop apps via Electron; mobile via React Native; CLI version available. Distinguishes itself from Notion / Evernote by being self-hostable and not gating features behind subscriptions.

## Key features

- Markdown-first notes with rich-text fallback.
- End-to-end encrypted sync via user-chosen backend (Joplin Cloud / Dropbox / OneDrive / Nextcloud / WebDAV / S3).
- Cross-platform: Windows, macOS, Linux, Android, iOS, CLI.
- Plugin ecosystem.
- Web Clipper browser extension.
- Note tagging, search, attachments, drawings (Excalidraw integration).
- License is `NOASSERTION` per SPDX — verify the LICENSE file in the repo before commercial redistribution; the core has been AGPL-ish historically.

## Tech stack

- TypeScript primary across the codebase.
- Electron for desktop.
- React Native for mobile.

## When to reach for it

- You want self-hostable, sync-friendly note-taking without Notion's lock-in.
- You're privacy-sensitive and need E2EE on notes.
- You want Markdown as the canonical format.

## When *not* to reach for it

- You want collaborative real-time editing — Joplin's sync is async / file-based.
- You want a polished, hosted UX without setup — Joplin Cloud exists but the appeal is self-hosting.

## Maturity signal

55k stars, 6k forks, actively maintained. The 571 open-issues count tracks the breadth of supported platforms + sync backends.

## Alternatives

- Obsidian — local-first Markdown notes; closed source but free for personal use.
- Logseq — OSS local-first outliner-style.
- Notion — managed SaaS, more powerful but cloud-locked.

## Tags

typescript, electron, react-native, note-taking, cross-platform, self-hosted, markdown, sync, encryption
