# papers-we-love/papers-we-love

The canonical curated index of computer-science research papers worth reading — the "what should I read in distributed systems / type theory / databases" reference for working engineers.

## What it is

A long-running community-maintained catalog of CS academic papers organized by topic — distributed systems, databases, programming languages, machine learning, security, networking, theory of computation, mathematics, and many more subfields. Each entry includes a brief annotation about why the paper matters and what to read it for. Companion to the paperswelove.org meetup network, which runs in-person paper-reading discussions in cities around the world. Used as the seed reading list for paper-reading clubs at countless tech companies.

## Key features

- Coverage across the major CS subfields with one curated entry per influential paper.
- Per-paper annotations explaining historical significance + what makes the paper worth the read time.
- Local meetup chapter network — many cities have in-person paper-reading clubs that announce monthly papers from this catalog.
- Tightly curated — open issues count near 2 is exceptional for a repo this size and reflects rigorous editorial standards.
- Multi-language translations of the meta documentation (intro + how-to-contribute) via community PRs.

## Tech stack

- Markdown content only — no build, no JS, no generator.
- Shell at the language tag covers the CI scripts that lint links and check formatting.
- The companion meetup site (paperswelove.org) is a separately-hosted property.

## When to reach for it

- You're an engineer wanting to read foundational CS papers but don't know where to start in your domain.
- You're a tech lead starting a paper-reading club and need a vetted starter list.
- You're a CS instructor curating supplementary reading for a course.
- You're a researcher needing pointers into adjacent subfields (e.g. a DB engineer wanting type-theory entry points).

## When *not* to reach for it

- You want recent / bleeding-edge papers — the canon here skews to historically influential papers. Use arXiv listings or Papers With Code for current research.
- You want implementation-paired papers — Papers With Code's annotations link directly to code; this list focuses on the papers themselves.
- You want a license-clean training corpus — SPDX is `null`; the curation list is reusable as fair use for educational purposes, but verify the LICENSE file before bulk redistribution.

## Maturity signal

107k stars, 6k forks, last push recent — actively curated for 11+ years. The high watcher count (3,122) reflects engineers using this as their canonical CS reading source. Open-issues count of 2 is exceptional and signals tight editorial triage rather than abandonment. The meetup chapter network adds a layer of community ownership that pure markdown lists rarely have.

## Alternatives

- arXiv directly — for recent papers in your domain (no annotations).
- Papers With Code — for ML papers with implementation links.
- ACM Digital Library / IEEE Xplore — for paywalled but more comprehensive coverage of CS conference proceedings.
- The Morning Paper (Adrian Colyer) — historical blog with daily annotated paper summaries.

## Notes

The "Papers We Love" framing is deliberate — entries pass an editorial filter of "papers we actually love and read again", not just "papers that were published." That's why the list stays curated and small rather than mushrooming into thousands of entries. License absence is the typical gotcha for community curation lists; preserve attribution downstream and you're generally safe.

## Tags

awesome-list, education, computer-science, papers, research, learn-to-code, academic, curated-directory
