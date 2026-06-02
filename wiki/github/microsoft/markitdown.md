# microsoft/markitdown

A Python tool from Microsoft that converts files (PDF, Office docs, audio, images, etc.) to Markdown — purpose-built for feeding documents into LLM pipelines.

## What it is

A document-to-markdown converter that handles a wide range of input formats: PDF, Word, PowerPoint, Excel, EPUB, HTML, images (via OCR), audio (via transcription), ZIPs (recursively), and more. Output is clean Markdown suitable for direct injection into LLM context windows or RAG pipelines. Built by Microsoft's AutoGen team as the document-ingestion piece for agentic workflows.

## Key features

- Multi-format input: PDF, DOCX, PPTX, XLSX, HTML, EPUB, audio (via Whisper / Azure Speech), images (via OCR or vision models), ZIP archives.
- Single-output format — clean Markdown that LLMs handle better than raw HTML/PDF.
- Integrates with AutoGen (Microsoft's agent framework) and LangChain.
- pip-installable (`pip install markitdown`) with optional extras for specific formats.
- CLI tool + Python library API.
- MIT-licensed.

## Tech stack

- Python primary.
- Optional dependencies for specific formats (PyMuPDF, python-docx, openpyxl, Whisper, etc.).
- Distributed via PyPI.

## When to reach for it

- You're building an LLM agent that needs to ingest documents in many formats and want one tool instead of N format-specific libraries.
- You're standing up a RAG pipeline and want clean Markdown chunks instead of dealing with PDF layout quirks.
- You're scripting bulk document conversion for archival or training data prep.

## When *not* to reach for it

- You need perfect-fidelity document conversion (e.g. PDF → DOCX preserving every visual element). Markitdown optimizes for readable Markdown, not visual fidelity.
- You want specialized PDF handling (tables, formulas, multi-column layouts) — purpose-built PDF tools (pdfplumber, marker, unstructured) may give better results for those edge cases.
- You're not in Python — port equivalents exist (e.g. for Node), but they're independent projects.

## Maturity signal

140k stars, 9.6k forks, MIT, last push 2026-05-26. 1.5-year-old project from Microsoft — fast-rising because the "documents → markdown for LLMs" problem was widely felt. Open-issues count of 796 tracks per-format edge cases (PDF layouts, Excel cell types, audio language detection) more than core defects. Microsoft stewardship signals stable maintenance.

## Alternatives

- `unstructured-io/unstructured` — use when you want more granular document-element extraction (titles, tables, lists separately).
- `Y2Z/marker` — use when you specifically want high-quality PDF-to-Markdown with layout preservation.
- Pandoc — use when you want general-purpose document conversion with format-flexibility on the output side.
- Format-specific libraries (PyPDF2, python-docx) — use when you need fine-grained control over one format.

## Notes

The "one tool, many formats" pitch is the value here. The 9-month sprint from launch to 140k stars reflects how acute the document-ingestion problem was for LLM pipelines. Microsoft's MIT license + AutoGen-team stewardship make this the safest "feed documents to your agent" choice for most projects.

## Tags

python, markdown, document-conversion, pdf, microsoft-office, large-language-model, retrieval-augmented-generation, command-line-interface, library
