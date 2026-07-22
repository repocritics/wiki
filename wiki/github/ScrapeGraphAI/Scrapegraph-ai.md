# ScrapeGraphAI/Scrapegraph-ai

A Python web scraping library that uses LLMs and graph-based pipelines to extract structured data from websites and local documents from a natural-language prompt.

## What it is

ScrapeGraphAI lets you describe the information you want and builds a scraping pipeline to extract it from a web page or a local file (XML, HTML, JSON, Markdown, and similar). Instead of hand-writing selectors, you supply a prompt and a source, and the library orchestrates LLM calls through a graph of steps to return structured output. It is aimed at developers who want prompt-driven, self-hosted extraction and control over which LLM they use.

## Key features

- Prompt-plus-source scraping through the `SmartScraperGraph` pipeline, returning structured dictionaries as output.
- A set of prebuilt pipelines: `SmartScraperGraph`, `SearchGraph`, `SpeechGraph`, `ScriptCreatorGraph`, and multi-page variants that parallelize LLM calls.
- Bring-your-own LLM across OpenAI, Groq, Azure, Gemini, MiniMax, and local models via Ollama.
- Works on both remote URLs and local documents, with browser fetching via Playwright.
- Integrations documented for LangChain, LlamaIndex, CrewAI, Agno, and low-code platforms, plus an MCP server.

## Tech stack

- Primary language: Python, requiring `>=3.12,<4.0`; distributed as `scrapegraphai` (version 2.1.6 in the manifest), built with hatchling.
- Built on LangChain (`langchain>=1.2.0`) with provider packages: `langchain-openai`, `langchain-mistralai`, `langchain-aws`, `langchain-ollama`, and `langchain_community`.
- Fetching and parsing via `playwright>=1.57.0`, `undetected-playwright`, `beautifulsoup4`, `html2text`, and `minify-html`.
- Supporting libraries include `tiktoken`, `semchunk`, `pydantic>=2.12.5`, `free-proxy`, `ddgs`, and the `scrapegraph-py` SDK.
- Optional extras: `burr` (workflow UI), `nvidia` endpoints, and `ocr` (surya-ocr, matplotlib, pillow).

## When to reach for it

- Extracting structured fields from pages or local files by describing them in natural language rather than writing parsers.
- Running scraping on your own infrastructure with a local LLM (Ollama) for privacy or cost control.
- Generating reusable scraping scripts or audio summaries via the ScriptCreator and Speech pipelines.
- Fanning out extraction across multiple pages or search results with the multi-page pipelines.

## When *not* to reach for it

- High-volume scraping that needs managed browsers, proxies, and anti-bot handling out of the box — the README points to the paid cloud API for that.
- Deterministic, high-throughput extraction where per-page LLM calls add latency and token cost.
- Projects pinned below Python 3.12, since the manifest requires 3.12+.
- Simple, well-structured sources where a direct parser (BeautifulSoup or Playwright alone) is cheaper and more predictable.

## Maturity signal

The library is actively maintained, with a recent last push, an MIT license, and a very low open-issue count for its audience size, which suggests issues are being triaged and closed. It recently moved onto the LangChain 1.x line and requires Python 3.12+, indicating the maintainers keep dependencies current rather than freezing them. Overall it reads as an established, actively developed OSS project with a companion commercial API.

## Alternatives

- Firecrawl — use it when you want a managed crawl-and-extract API rather than a self-hosted library (the repo lists itself as a Firecrawl alternative).
- Crawl4AI — use it when you want an open-source, LLM-friendly crawler focused on crawling and Markdown output.
- Playwright or BeautifulSoup directly — use these when the target is well-structured and you do not need LLM-driven extraction.
- The managed ScrapeGraphAI cloud API — use it when you want zero infrastructure, managed JS rendering and anti-bot, and built-in crawl and monitor jobs, billed per credit.

## Notes

The open-source library and the managed cloud API are distinct products: the library (`scrapegraphai`, MIT) runs on your own infrastructure with your own LLM keys, while the hosted service is used through separate `scrapegraph-py` and `scrapegraph-js` SDKs and is paid. Fetching website content requires running `playwright install` after pip installation. The library collects anonymous telemetry that can be disabled via `SCRAPEGRAPHAI_TELEMETRY_ENABLED=false`.

## Tags

python, web-scraping, data-extraction, llm, rag, ai-scraping, langchain, playwright, crawler, library, machine-learning
