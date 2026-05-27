# Workflow 2: Ingest

**Group.** Upstream
**Goal.** Create the right note type in the right folder with enrichment.

## Type detection first

Before any write, Hermes identifies the input type and routes to the correct pipeline:

| Input signal | Detected type | Destination folder |
| --- | --- | --- |
| DOI in `.bib` | Article / dataset / cited repo | `20-sources/01-literature/` |
| `github.com/...` or `gitlab.com/...` URL | Repository | `20-sources/02-items/` |
| PyPI / npm / CRAN URL | Package | `20-sources/02-items/` |
| Product or vendor URL | Product | `20-sources/02-items/` |
| Standards body URL (IEEE, RFC, W3C) | Standard | `20-sources/02-items/` |
| ORCID or person name | Person | `20-sources/03-entities/01-people/` |
| ROR or institution name | Organization | `20-sources/03-entities/02-organizations/` |
| Conference / journal name | Venue | `20-sources/03-entities/03-venues/` |
| DOI not in `.bib` | Unknown | **Prompt human to add to Zotero first** |

**Design rule.** The agent never creates a note in the wrong folder. If the type is ambiguous, it asks before proceeding.

## Steps

1. Hermes (researcher) detects the input type using the dispatch table above.
2. Routes to the correct folder and template.
3. Creates the note from the template.
4. Enriches metadata via type-specific APIs (see per-type table below).
5. Promotes stable identifiers (DOI, OpenAlex ID, ORCID, ROR) to main YAML.
6. For articles / preprints: Marker extracts the PDF to markdown, written to `90-assets/extracts/<citekey>.md`. The source-note's `extract_path` frontmatter is populated; `zotero_uri` and `pdf_uri` are populated from the Zotero item key. **The PDF itself is not copied into the vault** — it remains in Zotero's storage, reachable via `pdf_uri`.
7. Proposes `_draft_classification` for human review.
8. Adds cross-link candidates in the `Cites` / `Cited by` sections, updates indexes, writes the session log.
9. Writes stable IDs back to Zotero's `Extra` field via `pyzotero` (closes the loop).

## Per-type enrichment

| Detected type | Metadata source | Content extraction | Cross-link checks |
| --- | --- | --- | --- |
| Article / preprint | OpenAlex + Semantic Scholar + PubMed; PDF via Marker | Marker extracts full text to `90-assets/extracts/<citekey>.md`; summary in source-note body; PDF stays in Zotero. | Authors → person-notes; cited works → existing literature citekeys |
| Repository | GitHub API (REST or GraphQL) | `README.md` + `CHANGELOG.md` retrieved directly; summary into `What it is` | Primary maintainer → person-note; org owner → org-note |
| Package | PyPI / npm / CRAN API | Package description + project links | Maintainer → person-note; underlying repo → repo item-note |
| Product | Vendor URL via `generic-rest-bridge` or MarkItDown fallback | Homepage text → summary | Vendor org → org-note (if applicable) |
| Standard | Standards body URL (IEEE, RFC, W3C) | MarkItDown for HTML standards docs | Issuing org → org-note |
| Person | ORCID API + OpenAlex Authors | (none — entity note) | Co-authors derived from shared source-notes; affiliations → org-notes |
| Organization | ROR API + OpenAlex Institutions | (none — entity note) | Affiliated people → person-notes |
| Venue | OpenAlex Venues + DBLP | (none — entity note) | (none) |

**Content extraction fallback.** Marker handles PDFs; MarkItDown handles the long tail (HTML pages, Office documents, web standards). Neither is a Hermes skill — both are external CLIs the researcher invokes through the appropriate skill. Extracted markdown lands in `90-assets/extracts/<citekey>.md`; the source-note's `extract_path` frontmatter records the canonical location. Re-extraction is cheap (rerun Marker on the Zotero PDF, overwrite the extract file); the source-note is unaffected by re-extraction.

## Owners

Hermes for type detection, routing, creation, enrichment, content extraction, candidate links. Human resolves ambiguity and reviews the resulting note.

## Card lifecycle

`ready` (file-system watcher or `.bib` change creates `intake:source` card; watcher-triggered cards skip `pending`) → `active` (Researcher claims within 60s) → `awaiting-review` (Researcher exits with `_draft_classification` proposed) → operator advances to [#4 Triage](04-triage.md). Failed metadata fetch increments `retry_count`; threshold breach moves to `blocked-on-human` per [03-board.md retry pattern](../03-board.md#retry-pattern).

## Command

`hermes run llm-wiki ingest --source {citekey}`.

## Example

Citekey `mamykina2010sense` now exists in `library.bib`. The researcher runs `hermes run llm-wiki ingest --source mamykina2010sense` → DOI in `.bib` routes to `20-sources/01-literature/` → creates a note from `source-note.md` → Marker extracts the PDF to `90-assets/extracts/mamykina2010sense.md` → populates source-note frontmatter (`extract_path: 90-assets/extracts/mamykina2010sense.md`, `pdf_uri: zotero://open-pdf/library/items/<key>`, `zotero_uri: zotero://select/items/<key>`) → enriches via OpenAlex (citation count, abstract) and PubMed (publication metadata) → proposes `_draft_classification: { topic: [receptivity-detection], methods: [field-study] }` → writes the OpenAlex ID back to Zotero's `Extra` field → exits at `awaiting-review`. The human will triage it next.

## Related

- **Previous workflow:** [#1 Zotero capture](01-zotero-capture.md)
- **Next workflow:** [#4 Triage](04-triage.md)
- **Profile:** [profiles/researcher.md](../profiles/researcher.md)
