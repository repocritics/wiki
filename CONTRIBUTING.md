# Contributing to RepoCritics Wiki

Thanks for helping build the OSS knowledge base!

## How to Submit a PR

1. Find the wiki file for a repo: `wiki/{platform}/{owner}/{name}.md`
2. Click "Edit" on GitHub → make changes → submit PR.
3. Wait for automated review (usually < 1 hour).

## What's Accepted

✅ **Encouraged**:
- Typo fixes / grammar
- Broken link updates
- Production caveats ("Don't use X in Y environment")
- Historical context ("v2 was a major rewrite that broke...")
- Alternative library recommendations
- Performance benchmarks (with source links)
- Real-world usage stories

❌ **Rejected**:
- Marketing / promotional language
- Unverified claims without citation
- Personal attacks against maintainers
- Spam / off-topic content
- AI-generated text dumps (use the source repos for that)

## Review Layers

Your PR goes through 4 automated checks:

1. **AI Review (Sonnet)** — 30 seconds. Auto-merges typo/link/factual updates with ≥ 90% confidence.
2. **Trusted Editor Fast-Path** — If you have ≥ 2 years on GitHub + ≥ 5 public repos + ≥ 100 stars, your own PRs auto-merge after Layer 1.
3. **Community LGTM** — 3 trusted editors approving = merge.
4. **Human Review (Tom)** — Vandalism, legal issues, or political controversies escalate to Tom.

## Style Guide

- Markdown (CommonMark).
- 80 char soft wrap.
- Headings: `#` for repo name, `##` for sections.
- Sections: Overview / Getting Started / Production Notes / Alternatives / History / References.
- Cite sources with footnote links (`[^1]`).
- Be terse — readers and LLMs both prefer dense, citation-rich prose.

## Maintainer Response

If you're the maintainer of a repo and disagree with content on the wiki:

- ❌ **Don't** edit the wiki to praise your own project (we will revert).
- ✅ **Do** add a "Maintainer Response" footnote: cite issues, link clarifications, request corrections.
- For factual errors or libel, open an issue tagged `maintainer-correction` — Tom reviews these directly.

Per RepoCritics' [Michelin-guide principle](https://repocritics.com/about): subjects don't review themselves.

## License

By contributing, you agree your work is licensed under [CC-BY-SA 4.0](./LICENSE).

## Code of Conduct

See [CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md).
