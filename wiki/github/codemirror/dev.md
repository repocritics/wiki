# codemirror/dev

> The development monorepo and bug tracker for CodeMirror 6 — a coordination shell, not a package you install.

[GitHub repo](https://github.com/codemirror/dev) ·
[Official website](https://codemirror.net/) ·
License: NOASSERTION (packages are MIT[^1])

## Overview

`codemirror/dev` is the central development repository for CodeMirror, the in-browser code editor by Marijn Haverbeke[^2]. It is deliberately not a library you depend on: it holds the shared issue tracker, the `bin/cm.js` orchestration script, and the dev/build tooling that stitches together the roughly two dozen separately-published npm packages (`@codemirror/state`, `@codemirror/view`, `@codemirror/commands`, `@codemirror/language`, the `@codemirror/lang-*` family, and the underlying `@lezer/*` parser packages) that actually make up CodeMirror 6. If you want to *use* CodeMirror you install those packages from npm and ignore this repo entirely; if you want to *develop on* CodeMirror this is where you clone the constellation and wire it together[^3].

CodeMirror 6 (the target of this repo) is a ground-up rewrite of the widely deployed CodeMirror 5, released in 2021 with an entirely different architecture: functional, immutable state; a strict view/state separation; and extension composition via facets rather than a monolithic options object[^2]. The rewrite traded CodeMirror 5's easy drop-in configurability for correctness, accessibility, mobile/IME support, and a tree-shakeable module graph. This is the project's defining tension — CM6 is more principled and more capable than CM5, but assembling a working editor now means composing extensions rather than passing an options bag.

The most important operational fact about this specific repository: it is **archived on GitHub** (April 2026) and development has moved to Marijn Haverbeke's self-hosted forge at `code.haverbeke.berlin/codemirror/dev`[^4]. The GitHub presence is now a read-only mirror. The published npm packages are unaffected and continue to be maintained; only the location of the source and issue tracker changed.

## Getting Started

You almost never clone this repo. To *use* CodeMirror 6, install the packages:

```bash
npm install @codemirror/state @codemirror/view @codemirror/commands
# language + convenience bundle:
npm install codemirror @codemirror/lang-javascript
```

```js
import { EditorView, basicSetup } from "codemirror";
import { javascript } from "@codemirror/lang-javascript";

const view = new EditorView({
  doc: "console.log('hello')\n",
  extensions: [basicSetup, javascript()],
  parent: document.body,
});
```

To *develop on* CodeMirror (what this repo is for), you need Node.js 16 and the `cm.js` bootstrapper[^3]:

```bash
git clone https://github.com/codemirror/dev.git
cd dev
node bin/cm.js install   # clones every package, installs deps, builds
npm run dev              # rebuild-on-change + demo/tests server on :8090
```

## Architecture / How It Works

CodeMirror 6 is organized around a small number of hard boundaries:

- **`EditorState`** is immutable. You never mutate the document; you dispatch a **transaction** that produces a new state. Undo, collaboration, and time-travel all fall out of this.
- **`EditorView`** is the mutable DOM layer. It observes state and reconciles the visible DOM, including a viewport-based virtual rendering scheme so multi-megabyte documents stay responsive.
- **`Text`** is a rope (persistent tree of string chunks), which is why large-document edits and line lookups are cheap.
- **Facets** are the extension mechanism: an extension contributes values to named facets (keymaps, theming, gutters, decorations), and CodeMirror resolves them into effective configuration. Extensions compose; ordering and precedence are explicit.
- **Syntax** is delivered by **Lezer**[^5], a separate incremental GLR parser system also by Haverbeke. Grammars compile to compact parse tables; the parser reuses unchanged subtrees on edit, which is what makes highlighting fast enough to run on every keystroke.

The `dev` repo itself contributes none of these — it is the build coordinator. `bin/cm.js` clones each package into a sibling checkout, links them together, and provides `build` / `dev` / `test` commands so a contributor can work across package boundaries without publishing intermediate versions. Think of it as a hand-rolled monorepo tool predating (and lighter than) the current wave of workspace managers.

## Production Notes

- **Do not depend on this repository.** Nothing here is on npm. Pin the individual `@codemirror/*` packages instead. Because they version independently, keep `@codemirror/state` and `@codemirror/view` on compatible ranges — a mismatched `state`/`view` pair is the classic CM6 breakage and usually surfaces as duplicate copies of `state` in the bundle.
- **Single-instance state modules.** `@codemirror/state` must be deduplicated in your bundle; two copies produce cryptic "unrelated transaction" / facet errors. Use your bundler's dedupe/resolutions if a transitive dep drags in a second copy.
- **CM5 → CM6 is a rewrite, not an upgrade.** There is no migration path in the "bump the version" sense; the API surface, config model, and extension system are all different. Budget a real port. CM5 remains separately maintained for legacy consumers.
- **Assembly cost.** `basicSetup` gets you a usable editor quickly, but any non-trivial feature (custom gutters, autocomplete sources, linting, collaborative editing) means learning the facet/decoration/state-field model. The learning curve is front-loaded.
- **Archived GitHub repo.** Issues and PRs against `github.com/codemirror/dev` are effectively frozen; new bug reports belong on the Haverbeke forge or the CodeMirror discussion forum[^4]. CI badges and links pointing at GitHub Actions here are stale.
- **Node 16 bootstrap.** The `cm.js` install flow targets an older Node; contributors on modern Node versions occasionally hit build-script friction. This only affects people developing CodeMirror itself, not consumers.

## When to Use / When Not

**Use CodeMirror 6 (via its packages) when:**
- You need an embeddable code/text editor with real syntax awareness, strong accessibility, and good mobile/IME behavior.
- Bundle size matters and you want to include only the language modes and features you use.
- You want fine-grained programmatic control over document state, decorations, and transactions.

**Look elsewhere when:**
- You want a full IDE experience (IntelliSense, hover, multi-file) out of the box — Monaco is closer.
- You need a drop-in editor with minimal API surface and are porting legacy CodeMirror 5 code — the rewrite means real work.
- You want rich-text/WYSIWYG document editing rather than code — that is ProseMirror's domain, not this one.

## Alternatives

- microsoft/monaco-editor — the editor that powers VS Code; use it when you want IDE-grade features out of the box and can absorb the much larger bundle and heavier runtime.
- ajaxorg/ace — the older, mature Ace editor; use it when you need a stable options-bag API and broad legacy-browser support rather than CM6's compositional model.
- ProseMirror/prosemirror — same author, but for structured rich-text/document editing; use it when your content is prose with schema, not source code.
- codemirror/codemirror5 — the previous generation; use it only when you cannot afford the CM6 rewrite and need the simpler drop-in config.
- lezer-parser/lezer — the parser system underneath CM6; use it directly when you need incremental parsing without an editor UI.

## History

| Version | Date | Notes |
|---------|------|-------|
| CM5 line | pre-2021 | Original CodeMirror; still separately maintained for legacy use. |
| dev repo created | 2018-08-28 | Monorepo/coordination repo for the "codemirror.next" rewrite. |
| CodeMirror 6.0 | 2021-06 | Ground-up rewrite: immutable state, facets, Lezer parsing[^2]. |
| Ongoing 6.x | 2021–2026 | Independent per-package semver across `@codemirror/*` and `@lezer/*`. |
| GitHub archived | 2026-04 | Development moved to `code.haverbeke.berlin`; GitHub becomes a mirror[^4]. |

## References

[^1]: CodeMirror 6 packages are published under the MIT license; the GitHub API reports NOASSERTION for this coordination repo because it carries no root SPDX license file. https://github.com/codemirror/state/blob/main/LICENSE
[^2]: CodeMirror 6 system guide and design rationale (immutable state, view/state split, facets). https://codemirror.net/docs/guide/
[^3]: Repository README — development bootstrap via `node bin/cm.js install`, `npm run dev` on port 8090. https://github.com/codemirror/dev
[^4]: README notice: "This repository has moved to https://code.haverbeke.berlin/codemirror/dev"; the GitHub repo is archived. https://code.haverbeke.berlin/codemirror/dev
[^5]: Lezer parser system — incremental GLR parsing used for CodeMirror syntax highlighting. https://lezer.codemirror.net/

## Tags

javascript, code-editor, text-editor, browser, editor-component, codemirror, lezer, immutable-state, frontend, monorepo, archived
