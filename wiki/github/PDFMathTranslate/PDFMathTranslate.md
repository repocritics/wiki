# PDFMathTranslate/PDFMathTranslate

Command-line and GUI tool that translates scientific PDF documents while preserving their original layout, formulas, and formatting.

## What it is

PDFMathTranslate translates full-text PDFs into another language without breaking the document's structure. It targets people who read scientific and technical papers across a language barrier and need formulas, charts, tables of contents, and annotations to survive the translation. It produces both a monolingual translated PDF and a side-by-side bilingual PDF, and can be driven from the command line, a browser GUI, Docker, a Zotero plugin, or an MCP interface. The work was accepted as an EMNLP 2025 System Demonstration.

## Key features

- Preserves formulas, charts, table of contents, and annotations during translation.
- Emits two outputs per run: a translated `-mono.pdf` and a bilingual `-dual.pdf`.
- Supports multiple translation services selectable via `-s`, including Google (default), DeepL, Ollama, OpenAI, Azure, and Tencent.
- Layout detection uses a DocLayout-YOLO ONNX model to locate text regions before translation.
- Command-line options for partial-document translation, source/target languages, multi-threading, translation caching, batch directory translation, and custom prompts.
- Ships a browser GUI (Gradio), Docker images, and MCP support in both STDIO and SSE modes.

## Tech stack

- Python, constrained to `>=3.11,<3.13`.
- PDF handling via PyMuPDF (`<1.25.3`), pdfminer.six (`==20250416`), pikepdf, and fontTools.
- Layout detection via onnx / onnxruntime and opencv-python-headless, plus the `wybxc/DocLayout-YOLO-DocStructBench-onnx` model.
- Translation backends including openai (`>=1.0.0`), deepl, ollama, azure-ai-translation-text, tencentcloud-sdk-python-tmt, and xinference-client; optional argostranslate.
- babeldoc (`>=0.1.22,<0.3.0`) available as an experimental backend.
- GUI built on gradio (`<5.36`) with gradio_pdf (`>=0.0.21`).
- Optional extras: MCP support via mcp (`>=1.6.0`), a Flask/Celery/Redis backend, and CUDA/DirectML ONNX runtimes.
- Packaged with hatchling and distributed on PyPI as `pdf2zh` (manifest version 1.9.11); Docker images published to Docker Hub and GHCR.

## When to reach for it

- Reading foreign-language scientific or technical papers while keeping equations and layout intact.
- Producing bilingual, side-by-side PDFs for review or study.
- Batch-translating a directory of PDFs from the command line.
- Integrating PDF translation into a Python program (Python API) or an MCP-based agent workflow.

## When *not* to reach for it

- Environments that cannot run Python 3.11 or 3.12, since the package pins that range.
- Cases needing the additional edge-case handling, cross-column and cross-page consistency work, or web UI improvements, which live in the separate `PDFMathTranslate-next` fork rather than this repository.
- Regions with restricted network access, where downloading the required layout model can fail without the documented Hugging Face mirror workaround.
- Projects that cannot accommodate AGPL-3.0 licensing.

## Maturity signal

With over 35,000 stars, creation in September 2024, and a last push in May 2026, this is an actively maintained project with a steady stream of recent fixes. The README clarifies that version 2.0 moved to a separate organization repository (`PDFMathTranslate/PDFMathTranslate-next`), and positions this repository as the present, stable release of the original line. It is licensed AGPL-3.0.

## Alternatives

- BabelDOC — use it when you want the newer translation backend directly; it is the backend this project is migrating toward.
- PDFMathTranslate-next — use it when you need the additional features and edge-case handling, accepting that it is development-focused and not designed for community contributions.
- MathTranslate — use it when you specifically want the multi-threaded translation approach this project credits in its acknowledgements.

## Notes

- Two major forks coexist; the `-next` fork is explicitly intended for development only and does not address compatibility issues.
- The experimental v2.0 "precise" mode runs in an isolated environment and requires a separate provisioning step (`pdf2zh-setup-precise`) after install.
- Translation depends on downloading a DocLayout-YOLO ONNX model, which is a known pain point for users behind restrictive networks.

## Tags

python, pdf, translation, document-processing, cli, gui, mcp, llm, layout-analysis
