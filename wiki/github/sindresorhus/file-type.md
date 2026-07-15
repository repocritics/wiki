# sindresorhus/file-type

> Detect the binary file type of a file, stream, or byte buffer by reading its magic-number signature.

[GitHub repo](https://github.com/sindresorhus/file-type) ·
[License: MIT](https://github.com/sindresorhus/file-type/blob/main/license)

## Overview

`file-type` answers one narrow question: given some bytes, what kind of file is this? It ignores the filename extension entirely and instead reads the leading bytes — the [magic number](https://en.wikipedia.org/wiki/Magic_number_(programming)#Magic_numbers_in_files) — and matches them against a hand-maintained decision tree of signatures. It returns `{ext, mime}` (for example `{ext: 'png', mime: 'image/png'}`) or `undefined`. It is deliberately scoped to binary formats: it will not identify `.txt`, `.csv`, `.svg`, or other text-based formats out of the box, because those have no reliable byte signature[^1].

The package is one of Sindre Sorhus's long-lived utilities (first published 2014) and has become the default file-type sniffer in the Node.js ecosystem, pulled in by upload handlers, media pipelines, and CLI tools. The detection engine itself is built on Borewit's tokenizer stack (`strtok3` / `token-types`), and Borewit is effectively the maintainer of the internals today. That lineage matters: it is why `file-type` can detect a type from a partial stream or an HTTP range request without downloading the whole file.

The defining tension is honesty about what detection means. Magic-number matching is a best-effort hint, not validation and not a security boundary — a file can carry a valid PNG header and still be malformed or malicious, and many container formats (MP4, MOV, HEIC, DOCX, XLSX, EPUB) share the same leading bytes and are hard to disambiguate. The maintainers say so plainly and decline to treat detection-time resource exhaustion on untrusted input as a security issue[^2]. The other tension is packaging: since v17 the library is ESM-only, which is clean for modern projects and a recurring source of friction for CommonJS and TypeScript-CJS ones[^3].

## Getting Started

```sh
npm install file-type
```

```js
import {fileTypeFromFile} from 'file-type';

console.log(await fileTypeFromFile('Unicorn.png'));
//=> {ext: 'png', mime: 'image/png'}
```

```js
// From a buffer — works even if you only have the first few KB of a file.
import {fileTypeFromBuffer} from 'file-type';

const bytes = new Uint8Array([0x89, 0x50, 0x4e, 0x47]); // ‰PNG
console.log(await fileTypeFromBuffer(bytes));
//=> {ext: 'png', mime: 'image/png'}
```

Every entry point returns a `Promise`. Detection is async even for an in-memory buffer because the same tokenizer-based engine drives every source.

## Architecture / How It Works

`file-type` is a thin detection layer over a general-purpose byte tokenizer. Instead of loading a whole file into a `Buffer` and slicing it, it wraps the source in an `ITokenizer` (from `strtok3`) that exposes `peekBuffer` / `readBuffer` / `ignore` operations, then walks a large signature decision tree, peeking only as many bytes as each candidate format requires. Because the tokenizer can *seek* (fast-forward, and over HTTP even jump to the tail of a file), detection reads the minimum necessary — often only the first few bytes, sometimes a header plus a trailer.

All the public entry points funnel into this one engine:

- `fileTypeFromFile` (needs `node:fs`), `fileTypeFromBuffer` (`Uint8Array`/`ArrayBuffer`), `fileTypeFromBlob` (streams the Blob), and `fileTypeFromStream` (a web `ReadableStream`) are convenience wrappers.
- `fileTypeFromTokenizer` is the real primitive — it is what lets adapters like `@tokenizer/http` (HTTP range requests) and `@tokenizer/s3` supply bytes from remote sources without a full download.
- `FileTypeParser` is the class form. It carries options (`customDetectors`, an `AbortSignal`, `mpegOffsetTolerance`) and exposes `fromFile` / `fromBuffer` / `fromStream`.

The signature logic is a hand-written cascade of byte comparisons in the core module — not a declarative table — which is why format support grows by explicit PRs rather than configuration. Two families make this messy in practice. ISO base-media containers (MP4, M4A, MOV, 3GP, HEIC, AVIF) all begin with an `ftyp` box, so they are separated by inspecting the brand field; audio-vs-video disambiguation is imperfect enough that a dedicated `@file-type/av` detector exists. ZIP-based formats (DOCX, XLSX, PPTX, EPUB, APK, ODT) all start with the ZIP local-file-header signature, so the parser has to read into the archive's structure to tell them apart.

Extensibility is the `customDetectors` option: an array of `{id, detect(tokenizer, fileType)}` functions run *before* the built-ins. This is how text-based and niche formats are supported without bloating core — `@file-type/xml` (SVG, KML, RSS, XHTML), `@file-type/pdf`, and `@file-type/cfbf` (legacy Office / `.msi`) live as separate packages. One sharp edge: if a custom detector advances the tokenizer position and then returns `undefined`, no further detectors run and the result is `undefined`, because subsequent detectors can no longer see the consumed bytes.

## Production Notes

**ESM-only.** Since v17 there is no CommonJS build. `require('file-type')` throws. TypeScript projects on `"module": "commonjs"` cannot import it normally either; the documented workaround is `load-esm`, or converting the consuming project to ESM[^3]. Webpack users need a current version configured for ESM. This is the single most common integration complaint.

**Detection is a hint, not a gate.** The MIME it returns is derived from bytes an attacker controls. Do not use it as an allowlist for "safe" uploads and assume the file is benign or well-formed. For untrusted input the README recommends enforcing a size limit and running detection in a worker thread with a timeout (e.g. `make-asynchronous`); pathological inputs that cause slow reads are explicitly *not* treated as security vulnerabilities[^2].

**Sample size vs. accuracy.** `fileTypeStream` buffers a `sampleSize` (default 4100 bytes) to sniff before passing the stream through. Shrinking the sample lowers detection reliability; some formats need bytes beyond the default window, so a shorter sample can silently return `undefined` for a file that would otherwise be recognized.

**Shared-signature ambiguity is real.** If you rely on distinguishing MP4 from MOV, or DOCX from a plain ZIP, budget for the edge cases and consider the `@file-type/av` / `@file-type/cfbf` detectors. `mpegOffsetTolerance` exists specifically because real-world MP1/MP2/MP3/AAC files are often muxed with a slightly offset first frame; a tolerance of ~10 bytes recovers many technically-invalid-but-common files.

**Scope creep is refused, not accepted.** The maintainers only take PRs for common modern binary formats and ask you to open an issue first[^1]. If you need thousands of formats, text extraction, or content parsing, this library is intentionally the wrong tool.

## When to Use / When Not

**Use when:**
- You need to verify the actual type of an uploaded or downloaded binary regardless of its (untrusted) extension.
- You want to sniff a type from a stream, Blob, or remote URL without buffering the whole file.
- You're in a modern ESM Node, Deno, or bundled-browser environment and want a small, well-maintained dependency.

**Avoid when:**
- Your project is CommonJS and you can't adopt an ESM shim.
- You need text-format detection, deep metadata, or content extraction (use a heavier tool or a custom detector).
- You're treating the result as a security guarantee — it is not one.

## Alternatives

- apache/tika — JVM, server-side; detects and extracts text from thousands of formats. Use when you need breadth and content extraction, not a small JS dependency.
- file/file (libmagic) — the canonical Unix `file`/`libmagic` magic database. Use when you want a C library or CLI and the largest signature corpus.
- Borewit/music-metadata — same tokenizer lineage. Use when you need actual audio/video tags and technical metadata, not just the type.
- sindresorhus/image-type — a thin wrapper for the image-only subset. Use when you only care about images and want an even smaller surface.
- jshttp/mime-db — maps extensions to MIME types. Use when you already trust the filename and just need the mapping, not content sniffing.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014 | Initial release; simple leading-byte checks on a Node `Buffer`. |
| 16.x | ~2020 | Tokenizer-based streaming rewrite on `strtok3`; incremental reads and stream support. |
| 17.x | ~2022 | Converted to ESM-only; no CommonJS build[^3]. |
| 18.x–21.x | 2023–2025 | Move toward `Uint8Array`/web streams for cross-runtime use; `customDetectors` plugin API; split-out `@file-type/*` detectors. |

Version-to-date mappings above 16.x are approximate; consult the changelog for exact release dates.

## References

[^1]: file-type README — scope statement ("for detecting binary-based file formats, not text-based formats"; contributions only for common modern formats). https://github.com/sindresorhus/file-type#readme
[^2]: file-type README — detection is a best-effort hint, and robustness against malformed input is best-effort with worker-thread + size-limit guidance for untrusted files. https://github.com/sindresorhus/file-type#readme
[^3]: Sindre Sorhus, "Pure ESM package" notes — rationale and CommonJS migration guidance. https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c

## Tags

javascript, nodejs, file-type-detection, magic-numbers, mime, binary, esm, streams, uint8array, upload-validation
