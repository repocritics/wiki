# google/langextract

A Python library that extracts structured information from unstructured text with LLMs, mapping every extraction back to its exact source location.

## What it is

LangExtract is a Python library that uses LLMs to pull structured data out of unstructured text based on user-defined instructions and a few high-quality examples. It processes materials such as clinical notes, reports, or long documents, identifying and organizing key details while keeping each extracted item tied to the source text. It is aimed at developers who need reliable, traceable extraction across arbitrary domains without fine-tuning a model. Results can be saved as JSONL and rendered as an interactive HTML visualization for review.

## Key features

- Precise source grounding: every extraction is mapped to its character interval in the source text, and items that cannot be located are flagged with `char_interval = None` so they can be filtered out.
- Schema-consistent structured output driven by few-shot examples, using controlled generation on supported models like Gemini; enum and other constraints available via `output_schema`.
- Handling for long documents through text chunking, parallel processing (`max_workers`), and multiple extraction passes (`extraction_passes`) to improve recall.
- A self-contained interactive HTML visualization generated from the results file for reviewing large numbers of entities in context.
- Flexible model support: Google Gemini, OpenAI models (optional `openai` extra), and local models via Ollama, selected automatically by model ID.
- A plugin system for custom providers via entry points, plus opt-in Vertex AI and OpenAI Batch APIs for large-scale, cost-sensitive workloads.

## Tech stack

- Python, requiring `>=3.10`; packaged as `langextract` on PyPI (version 1.6.0 in this manifest).
- Core dependencies include `google-genai>=1.39.0`, `google-cloud-storage>=2.14.0`, `aiohttp>=3.8.0`, `numpy`, `pandas`, `pydantic>=1.8.0`, `regex`, `requests`, and `tqdm`.
- OpenAI support is an optional extra (`openai>=1.50.0`); dev tooling uses `pyink`, `isort`, `pylint`, `pytype`, and `tox`.
- Provider discovery via `langextract.providers` entry points for `gemini`, `ollama`, and `openai`.
- Import-linter contracts enforce module boundaries (for example, core must not import providers). Docker build supported. Apache-2.0 licensed.

## When to reach for it

- You need structured extraction (entities, attributes, relationships) from text where traceability to the exact source span matters.
- You work in a specialized domain (clinical, legal, literary) and want to define the task with a few examples instead of training a model.
- You are processing long documents and need chunking, parallelism, and multiple passes to maintain recall.
- You want to visually review and verify large extraction result sets in context.

## When *not* to reach for it

- You want extraction without an LLM; LangExtract's grounding and structured outputs depend on a model, and cloud models require an API key.
- You need guaranteed correctness; the README notes accuracy depends on the chosen LLM, prompt clarity, and examples, and models may occasionally pull content from the examples rather than the input.
- You want an officially supported and warranted Google product; the README explicitly states it is not one.
- Your task is simple pattern matching well served by regex or a rules engine, where an LLM adds unnecessary cost and latency.

## Maturity signal

LangExtract has over 37,000 stars since its July 2025 creation and a most-recent push in July 2026, indicating strong adoption and active maintenance, with a passing CI badge and a versioned PyPI release (1.6.0). The Apache-2.0 license, DOI/Zenodo citation, community-provider registry, and enforced import-boundary contracts point to a maintained, well-structured library — though the explicit "not an officially supported Google product" disclaimer means support is best-effort rather than commercial.

## Alternatives

- Instructor — use it instead when you mainly want Pydantic-typed structured outputs from LLM calls without source-span grounding or visualization.
- LlamaIndex — use it instead when extraction is one part of a broader retrieval and indexing pipeline.
- spaCy — use it instead when you want fast, deterministic, model-based NER without depending on an LLM per document.

## Notes

A distinctive design point is the emphasis on grounding as a first-class, checkable property: extractions that cannot be aligned to the source text are surfaced with a null character interval rather than silently returned, and the README recommends filtering on `char_interval` to keep only grounded results. The same JSONL output feeds directly into the interactive HTML visualizer, and batch processing is opt-in with automatic fallback to realtime calls below a configured prompt-count threshold.

## Tags

python, llm, information-extraction, nlp, structured-data, named-entity-recognition, few-shot-learning, gemini, library
