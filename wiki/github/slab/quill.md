# slab/quill

Quill — a modern WYSIWYG editor built for compatibility and extensibility. Cross-browser, cross-framework, with a powerful delta-based document model.

## What it is

A TypeScript / JavaScript rich-text editor library used in many production apps. Distinguishes itself via a "Delta" data model — every document change is a Delta operation, making collaborative editing and history tractable. Originally authored at Salesforce, now maintained by Slab. Used by Microsoft Teams, Figma, Slack, LinkedIn, and many others in production.

## Key features

- Delta-based document model (operational transformations friendly).
- Cross-framework — works with React, Vue, Angular, vanilla JS.
- Module-based extensibility.
- Themes (Snow, Bubble) + custom theming.
- Toolbar, syntax highlighting, table support.
- BSD-3-Clause / MIT licensed (verify per release).

## Tech stack

- TypeScript primary.
- npm package: `quill`.

## When to reach for it

- You need a robust WYSIWYG editor with a clean data model.
- You're building collaborative editing on top — Quill's Delta integrates with OT engines.

## When *not* to reach for it

- You want a markdown-flavored editor (Tiptap, ProseMirror direct).
- You want a fully prebuilt UI — Quill is library-style; you compose the surrounding UI.

## Maturity signal

44k+ stars, actively maintained.

## Alternatives

- Tiptap — ProseMirror-based modern alternative.
- ProseMirror directly — for full control.
- Slate.js — React-flavored framework.

## Tags

typescript, javascript, editor, wysiwyg, rich-text, library
