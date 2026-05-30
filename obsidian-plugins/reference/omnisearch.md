---
mode: how-to
audience: operator
topic: plugins
---

# omnisearch — evaluated, not depended on

Omnisearch adds fast fuzzy full-text search across the vault (filenames, note bodies, and optionally PDF text). It is a genuinely useful **human** convenience, but Memoria's design **does not depend on it**: agent-side search goes through Hermes tools and the `qmd` retrieval substrate (see [workflows/README.md](../../workflows/README.md)), not the Obsidian plugin — so the [Librarian](../../profiles/librarian.md) behaves identically whether or not Omnisearch is installed.

It lives in `reference/` rather than `recommended/` for one reason: its load-bearing settings — index depth, excluded folders, whether to index PDF text — are **workflow-personal, not design-standard**. Memoria ships no config for it and makes no claim on how it should be tuned.

- **Install it if** your vault is large enough that core search feels slow and you want fast human-driven lookup. Tune indexing to taste.
- **Don't wire it into agent workflows** — that's what the Hermes retrieval tools are for, and their results are auditable in a way a plugin's private index is not.
