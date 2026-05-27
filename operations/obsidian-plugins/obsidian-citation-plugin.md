# obsidian-citation-plugin

The current Required plugin for the Zotero ↔ vault bridge. Reads the BibTeX/BibLaTeX export Better BibTeX produces, exposes `@`-autocomplete for inserting citations, and creates literature notes from a configured template.

**Canonical template:** shipped at `.obsidian/plugins/obsidian-citation-plugin/data.json` in the starter vault — installed as the operator's working file directly, no template-copy step. The shipped file embeds the canonical source-note frontmatter (matching the schema in [05-notes-folders.md](../../05-notes-folders.md)), the `_draft_classification` block, and the `_enrichment` block. Treat changes to this file as schema migrations.

Load-bearing settings:

- `citationExportPath` — absolute path to the Better BibTeX export. Convention: name it after the vault (`00-meta/<vault-name>.bib`), not a generic `library.bib`. Memoria's vault uses `00-meta/memoria.bib`; research-wiki uses `00-meta/research-wiki.bib`.
- `citationExportFormat: "biblatex"` — the modern BibLaTeX format; preserves Unicode and modern entry types better than legacy `bibtex`.
- `literatureNoteFolder` — must point at the source-note literature folder (`20-sources/01-literature/` in Memoria's schema). Wrong folder means the plugin creates notes outside the Researcher's scope, and the linter immediately flags them.
- `literatureNoteTitleTemplate: "@{{citekey}}"` — keeps the filename and citekey identical so backlinks resolve cleanly.
- `literatureNoteContentTemplate` — the body skeleton. **This is the most load-bearing setting.** It must include:
  - The full canonical frontmatter (see [05-notes-folders.md](../../05-notes-folders.md) for the schema and key order).
  - The `_draft_classification` HTML comment block — Researcher fills it on ingest; human promotes to main frontmatter on triage.
  - The `_enrichment` HTML comment block — Researcher populates from API enrichment (citation count, scite, MeSH, OpenAlex concepts, etc.) and never touches main frontmatter for derived metrics.

The template is the source of truth for what a freshly-ingested literature note looks like. Drift between the template and the Researcher's expected schema breaks the triage and enrichment workflows. Treat changes to it as schema migrations — diff the change against [05-notes-folders.md](../../05-notes-folders.md) and the Researcher's contract in [researcher.md](../../profiles/researcher.md) before saving.

## Alternative: zotero-integration (Zotero Integration by mgmeyers)

A third option, not in current use but worth naming. Lives between citation plugin (BibTeX-file based) and [ZotLit](zotlit.md) (SQLite-direct): connects to Zotero's local HTTP API while Zotero is running, supporting color-coded annotation imports and Nunjucks templates.

- **Switch to this** if the annotation workflow ever moves from Obsidian (via PDF++) into Zotero itself. Color-coded highlights pulled from Zotero into structured literature notes is the use case it wins on.
- **Don't switch yet** because Memoria's design assumes PDF annotation happens in Obsidian, not Zotero — see [pdf-plus](pdf-plus.md). The decision to flip annotation workflow direction is bigger than a plugin choice; it changes the operator's daily rhythm.
