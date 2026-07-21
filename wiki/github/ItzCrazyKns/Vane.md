# ItzCrazyKns/Vane

Vane is a self-hosted AI answering engine that searches the web and returns cited answers using local or cloud language models.

## What it is

Vane is a web application that combines a metasearch backend with a large language model to answer questions and cite its sources. It is aimed at users who want a Perplexity-style answering experience but hosted on their own hardware, so that queries and search history stay private. It runs the whole stack locally, pairing a bundled SearxNG search engine with a model provider of the user's choice.

## Key features

- Works with local models through Ollama and with cloud providers including OpenAI, Anthropic Claude, Google Gemini, and Groq, and lets you mix models.
- Three search modes named Speed, Balanced, and Quality to trade latency against answer depth.
- Selectable source types (web, discussions, academic papers) and the option to restrict a search to specific domains.
- Web search runs through SearxNG so queries are proxied across multiple engines; image and video search are also supported.
- File uploads (PDFs, text files, images, and other documents) that can then be queried.
- Inline widgets for things like weather, calculations, and stock prices, plus a Discover feed, typed suggestions, and locally stored search history.

## Tech stack

- TypeScript, built on Next.js `^16.0.7` with React `^18` and Tailwind CSS `^3.3.0`.
- SQLite storage via Drizzle ORM `^0.45.2` and better-sqlite3 `^11.9.1`, with migrations managed by drizzle-kit `^0.18.1`.
- Model provider clients: `openai` `^6.9.0`, `@google/genai` `^1.34.0`, `ollama` `^0.6.3`, and `@huggingface/transformers` `^3.8.1`.
- Document ingestion through `pdf-parse` `^2.4.5`, `mammoth` `^1.9.1`, `officeparser` `^6.0.7`, `@mozilla/readability`, and `jsdom`.
- Supporting libraries: `zod` `^4.1.12` for validation, `js-tiktoken` for token counting, `mathjs` and `yahoo-finance2` for widgets, `playwright` `^1.59.1`, and `axios`.
- SearxNG as the search backend, deployed via Docker and docker-compose (a bundled image and a slim image that reuses an external SearxNG instance).

## When to reach for it

- You want a private, self-hosted alternative to hosted AI answer engines, with searches kept on your own machine.
- You already run, or want to run, local models through Ollama and prefer to keep inference off third-party APIs.
- You want answers that link back to cited web sources rather than ungrounded model output.
- You want to query your own uploaded documents alongside web results.
- You want to wire an AI search box into your browser or call it from your own application through its search API.

## When *not* to reach for it

- You do not want to operate infrastructure. Vane expects you to run Docker (or Node plus a SearxNG instance) and configure providers and API keys yourself.
- You need multi-user access control today. Authentication is listed as an upcoming feature, not a shipped one.
- You want a turnkey hosted product with no setup, in which case a managed service fits better.
- You need Tavily or Exa as the search source now; the README lists these as coming soon, with SearxNG as the current backend.

## Maturity signal

Vane is actively maintained. It carries roughly 35,800 stars and about 3,900 forks, was created in April 2024, and had commits pushed as recently as April 2026, with a current package version of 1.12.2. It is not archived and is MIT licensed. The 337 open issues are consistent with an active project that has a sizable user base rather than a sign of abandonment. The dependency set tracks recent releases (for example Next.js 16 and Zod 4), which points to ongoing upkeep.

## Alternatives

- Perplexity AI — use instead when you want a polished hosted product and do not care about self-hosting or keeping data local.
- SearXNG — use instead when you only need private metasearch and do not want an LLM answering layer on top.
- Morphic — use instead if you want another open-source AI answer engine and prefer its stack or hosting model.
- Khoj — use instead when the priority is searching and chatting over your own documents and notes more than answering general web queries.

## Notes

The repository's topics, one-click deploy templates, and the sponsor link all reference "perplexica" (for example the Sealos and Warp URLs use a `perplexica` template name), indicating Vane is a rebrand or successor of the author's earlier Perplexica project. The self-hosted setup has a hard dependency detail that is easy to miss: an external SearxNG instance must have JSON output and the Wolfram Alpha engine enabled, otherwise search and some widgets will not work.

## Tags

typescript, nextjs, react, ai-search-engine, answering-engine, llm, rag, self-hosted, searxng, ollama, sqlite, docker
