# opendataloader-project/opendataloader-pdf

An open-source PDF parser that extracts AI-ready structured data and auto-tags untagged PDFs for accessibility.

## What it is

OpenDataLoader PDF converts PDF files into Markdown, JSON (with bounding boxes), and HTML for use in RAG and LLM pipelines, and also auto-tags untagged PDFs into screen-reader-ready Tagged PDFs. It serves two audiences: teams building document ingestion for AI systems who need structured output with element coordinates, and organizations that must remediate PDFs to meet accessibility regulations. The core runs locally, with an optional hybrid mode that routes complex pages to a local AI backend.

## Key features

- Deterministic local extraction to Markdown, JSON, HTML, plain text, and annotated PDF, with a bounding box and semantic type for every element.
- XY-Cut++ reading-order analysis for multi-column and mixed layouts, plus header, footer, and watermark filtering.
- Hybrid mode that routes complex pages (borderless tables, scanned content, LaTeX formulas, chart descriptions) to a local AI backend; OCR supports 80+ languages.
- Auto-tagging of untagged PDFs into Tagged PDF under Apache 2.0; PDF/UA-1 and PDF/UA-2 export and an accessibility studio are enterprise add-ons.
- AI safety filtering that strips hidden text, off-page content, and suspicious invisible layers, with an optional `--sanitize` step for sensitive data.
- SDKs for Python, Node.js, and Java, plus a LangChain document loader integration.

## Tech stack

- Primary language: Java (core), requiring Java 11+; distributed via Maven Central as `org.opendataloader:opendataloader-pdf-core`.
- Python SDK (`opendataloader-pdf`, requires Python 3.10+) and Node.js SDK (`@opendataloader/pdf`), each invoking the JVM core.
- Organized as a monorepo workspace (root `package.json` with build/export/schema-generation scripts).
- Hybrid backend integrates a docling-based AI path and uses SmolVLM (256M) for picture and chart descriptions.

## When to reach for it

- Building RAG ingestion that needs element-level bounding boxes for source citations and click-to-source UX.
- Converting PDFs to clean Markdown or structured JSON for LLM context and chunking.
- Processing documents entirely on-premises without sending data to the cloud, including in legal, healthcare, and financial contexts.
- Remediating large volumes of untagged PDFs toward accessibility compliance (EAA, ADA/Section 508, and similar).

## When *not* to reach for it

- Parsing non-PDF office formats such as Word, Excel, or PowerPoint, which are explicitly not supported.
- Needing certified PDF/UA-1 or PDF/UA-2 output or a visual tag editor without an enterprise arrangement, since those are paid add-ons.
- Workloads that call `convert()` repeatedly on single files, because each call spawns a JVM process; the tooling expects batched input.
- Environments without a Java runtime available, since the core depends on Java 11+.

## Maturity signal

Despite being created in mid-2025, the project has grown quickly and is under heavy active development, with a last push within the last day and a large number of newly landed features across hybrid mode, OCR, and accessibility. It is Apache-2.0 for the core with a clearly delineated enterprise tier, and it publishes to PyPI, npm, and Maven Central, suggesting a maintained multi-language distribution. The relatively high open-issue count is consistent with a young, fast-moving project rather than one in maintenance mode.

## Alternatives

- docling — use it when an MIT license matters more than bounding boxes for every element or built-in AI safety filtering, which the README notes docling lacks.
- marker — use it when you already run on GPU and can accept much slower per-page throughput.
- pymupdf4llm — use it when speed matters most and lower table and heading accuracy is acceptable.
- unstructured — use it when you want a broad Apache-2.0 ingestion toolkit across many document types rather than a PDF-focused parser.

## Notes

The accessibility work is developed in collaboration with the PDF Association and Dual Lab (the veraPDF developers), follows the Well-Tagged PDF specification, and is validated programmatically with veraPDF. The README states this is the first open-source tool to generate Tagged PDFs end to end without relying on a proprietary SDK for the tag-writing step. Two behavioral caveats are documented: batch all files in a single `convert()` call because each call spawns a JVM, and `--use-struct-tree` takes precedence over `--hybrid` on tagged PDFs, so the hybrid backend is skipped if both are set.

## Tags

java, python, nodejs, pdf, document-parsing, ocr, accessibility, pdf-accessibility, rag, cli, library, machine-learning
