# RepoCritics Wiki

Community-maintained wiki for open source repositories.

> Part of [RepoCritics](https://repocritics.com) — pre-processed OSS knowledge for developers + AI agents.

## Structure

```
wiki/
├── github/{owner}/{name}.md
├── gitlab/{owner}/{name}.md
├── gitlab/{instance-url-encoded}/{owner}/{name}.md  ← self-hosted
├── gitee/{owner}/{name}.md
├── bitbucket/{owner}/{name}.md
├── sourceforge/_/{name}.md
├── svn/{host}/{path-encoded}.md
└── ...
```

Each `.md` file is the canonical wiki for one repository.

## How to Contribute

See [CONTRIBUTING.md](./CONTRIBUTING.md).

**Quick path**: Click "Edit Wiki" on the [RepoCritics](https://repocritics.com) page of any repo → GitHub edit page → submit PR.

## Review Process

PRs go through 4-layer automated review:

1. **AI auto-review** (Sonnet) — typo / link / factual updates auto-merged.
2. **Trusted Editor** — verified GitHub accounts (≥ 2 years, ≥ 5 repos, ≥ 100 stars) merge own PRs immediately.
3. **Community LGTM** — 3 trusted editors LGTM → auto-merge.
4. **Tom escalation** — vandalism / legal / political controversies.

## License

All wiki content is licensed under [CC-BY-SA 4.0](./LICENSE).

Code in `.github/workflows/` is MIT licensed.

## Status

🚧 Bootstrapping — initial seed in progress (Tom's fork list).

## Links

- [RepoCritics web](https://repocritics.com)
- [Issue tracker](https://github.com/repocritics/repocritics) (private, public issues coming Phase 2)
- [Discussions](https://github.com/repocritics/wiki/discussions)
