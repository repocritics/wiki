# microsoft/monaco-editor

> The code editor extracted from VS Code, packaged to run standalone in a browser.

[GitHub repo](https://github.com/microsoft/monaco-editor) ·
[Official website](https://microsoft.github.io/monaco-editor/) ·
[License: MIT](https://github.com/microsoft/monaco-editor/blob/main/LICENSE.txt)

## Overview

Monaco is the editor component from Visual Studio Code, generated straight from
VS Code's own sources with shims that let it run in a browser outside the desktop
app[^1]. It is not a separate editor with a shared lineage — it is the same code,
which is why it feels identical to VS Code's text surface (same keybindings,
same minimap, same IntelliSense widget, same diff view). Microsoft has shipped it
as a standalone npm package (`monaco-editor`) since 2016.

The defining consequence of that origin is the central tension of using Monaco:
you get a genuinely desktop-grade editor for free, but you inherit a component
that was designed for an Electron/desktop host, not for the open web. It leans on
Web Workers for language processing, ships a large bundle, has no official mobile
support[^2], and its integration story with bundlers is the single most common
source of setup pain. The public API surface is narrow and stable — only
`monaco.d.ts` is versioned; everything else is treated as private and may break
between releases[^3]. As of 2026 the package remains pre-1.0 (0.x versioning),
which reflects that private-internals stance rather than immaturity.

Monaco is best understood as an editor *view* plus a set of *language providers*,
not a full IDE. It renders and edits text extremely well; wiring it to real
language intelligence (via LSP, remote services, or the bundled TS/JSON/CSS/HTML
workers) is on you.

## Getting Started

```bash
npm install monaco-editor
```

The package ships an ESM build under `/esm` (for bundlers like webpack/Vite) and
a deprecated AMD build kept for backwards compatibility[^4]. Language features run
in Web Workers, so you must tell Monaco how to load them — this is the step most
first-time setups miss. With Vite:

```js
import * as monaco from 'monaco-editor';
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker';
import tsWorker from 'monaco-editor/esm/vs/language/typescript/ts.worker?worker';

self.MonacoEnvironment = {
  getWorker(_, label) {
    if (label === 'typescript' || label === 'javascript') return new tsWorker();
    return new editorWorker();
  },
};

monaco.editor.create(document.getElementById('container'), {
  value: 'function hello() {\n\treturn "world";\n}',
  language: 'javascript',
});
```

Without the worker wiring the editor still renders and edits text, but completion,
hovers, and diagnostics silently do nothing.

## Architecture / How It Works

Four concepts carry most of the API[^5]:

- **Models** hold content. A model is an opened "file" (real or virtual): text,
  language, and edit history. Models are the thing you mutate.
- **URIs** identify models — no two models may share a URI. A model created
  without one gets `inmemory://model/N`. URIs matter for correctness, not just
  bookkeeping: TypeScript import resolution and JSON-schema association key off
  the model URI, so a virtual filesystem layout (`file:///src/...`) is expected.
- **Editors** are the DOM-attached view of a model — view state, actions,
  commands. One model can be shown in several editors.
- **Providers** supply smart features (completion, hover, definitions). They
  operate on models and frequently mirror Language Server Protocol concepts,
  though Monaco's provider API predates and is distinct from LSP.

Heavy language work runs off the UI thread in **Web Workers**. Monaco bundles
full workers for TypeScript/JavaScript, JSON, CSS, and HTML — the TS worker is
effectively the TypeScript compiler running in-browser, which is why type-aware
completion works with no server. Everything else (syntax highlighting for other
languages) comes from **Monarch**, a declarative state-machine tokenizer defined
in JSON-like config[^6]. Note that Monaco does *not* use TextMate grammars out of
the box; matching VS Code's exact highlighting requires bolting on
`vscode-textmate` + `vscode-oniguruma` yourself[^7].

Because the source is mechanically derived from VS Code, upstream refactors flow
downstream. The stable contract is deliberately kept to `monaco.d.ts`; reaching
into internal modules (common when integrating deeply) means accepting breakage
on upgrades. `.dispose()` appears throughout — models, editors, providers, and
event subscriptions all need explicit disposal or they leak.

## Production Notes

- **Bundler configuration is the top footgun.** The workers must be emitted and
  resolvable at runtime. Community plugins exist for each toolchain
  (`monaco-editor-webpack-plugin`, `vite-plugin-monaco-editor`), and getting
  worker paths / cross-origin loading wrong yields the classic "Could not create
  web worker" failure. Loading from `file://` cannot create workers at all — a
  local HTTP server is required[^2].
- **Bundle size is large.** A full build pulls in the TS worker (the whole
  TypeScript compiler). Shipping only the languages you use, via the webpack
  plugin's `languages` allowlist or manual feature imports, is close to mandatory
  for web apps that care about payload.
- **No mobile support.** Touch input, virtual keyboards, and small viewports are
  explicitly unsupported[^2]. Do not ship Monaco as the editing surface of a
  mobile web app.
- **Private internals move.** Pinning an exact `monaco-editor` version and
  reading release notes before bumping is the safe path, especially if you touch
  anything outside `monaco.d.ts`.
- **VS Code version ≠ Monaco version.** They are decoupled; a Monaco release
  reflects the source at build time, not a VS Code release number[^8]. You cannot
  assume a given VS Code feature is present.
- **VS Code extensions do not run.** Extensions authored for the VS Code
  extension host will not work in Monaco; only pure-LSP language servers can be
  bridged (via `monaco-languageclient`)[^9].

## When to Use / When Not

**Use when:**
- You want the exact VS Code editing experience (keybindings, IntelliSense UI,
  diff editor, minimap) in a web app.
- You are building an in-browser IDE, playground, or config editor and can afford
  the bundle and worker setup.
- You need in-browser TypeScript/JSON/CSS/HTML intelligence with no backend.

**Avoid when:**
- The target is mobile or touch-first.
- You need a small, embeddable input for short snippets — the payload is
  disproportionate.
- You want a simple integration with minimal build config; the worker wiring is
  real overhead.
- You need TextMate-grammar-accurate highlighting without extra libraries.

## Alternatives

- codemirror/codemirror5 (and the CodeMirror 6 rewrite) — lighter, modular,
  first-class mobile/touch support; use when bundle size or mobile matters more
  than VS Code parity.
- ajaxorg/ace — mature, framework-agnostic, smaller footprint; use for a
  straightforward embeddable editor without a worker-based language layer.
- suren-atoyan/monaco-react (`@monaco-editor/react`) — not an alternative engine
  but the standard React wrapper that handles loader/worker setup; use with
  Monaco when your app is React.
- CodinGame/monaco-vscode-api — extends Monaco with real VS Code services and
  extension APIs; use when standalone Monaco is too limited and you need closer
  VS Code fidelity.
- TypeFox/monaco-languageclient — bridges Monaco to external LSP servers; use
  when you need language intelligence beyond the bundled TS/JSON/CSS/HTML workers.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016 | Standalone `monaco-editor` published from VS Code sources[^1]. |
| 0.x | 2016–2026 | Long-lived pre-1.0 line; only `monaco.d.ts` treated as stable API[^3]. |
| — | ~2023–2025 | ESM promoted as primary build; AMD build deprecated and slated for removal[^4]. |

Exact 0.x release dates and per-version changes are tracked in the repository's
changelog and GitHub releases rather than reproduced here to avoid inaccuracy[^8].

## References

[^1]: monaco-editor README, "FAQ — relationship between VS Code and the Monaco Editor." https://github.com/microsoft/monaco-editor#faq
[^2]: monaco-editor README, "FAQ — mobile support / web workers / file:// scheme." https://github.com/microsoft/monaco-editor#faq
[^3]: monaco-editor README, "Installing — `monaco.d.ts` specifies the API; everything else is private." https://github.com/microsoft/monaco-editor#installing
[^4]: monaco-editor README, AMD build deprecation notice. https://github.com/microsoft/monaco-editor#installing
[^5]: monaco-editor README, "Concepts — Models / URIs / Editors / Providers / Disposables." https://github.com/microsoft/monaco-editor#concepts
[^6]: Monarch tokenizer playground and docs. https://microsoft.github.io/monaco-editor/monarch.html
[^7]: monaco-tm — TextMate grammar support for Monaco via vscode-textmate/vscode-oniguruma. https://github.com/bolinfest/monaco-tm
[^8]: monaco-editor README, "FAQ — VS Code and Monaco versions are unrelated." https://github.com/microsoft/monaco-editor#faq
[^9]: monaco-languageclient — LSP client for Monaco. https://github.com/TypeFox/monaco-languageclient

## Tags

javascript, typescript, code-editor, browser, vscode, web-workers, monaco, ide, frontend, text-editor
