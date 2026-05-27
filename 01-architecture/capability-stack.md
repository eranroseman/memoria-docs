# Capability stack

The minimum capability stack to operate Memoria:

1. **Hermes**, configured with seven profiles (researcher, cartographer, socratic, writer, verifier, coder, linter).
2. **The Hermes native Kanban** with the required state machine and schema fields (see [03-board.md](../03-board.md)).
3. **Obsidian** as the vault editor and Dataview query host.
4. **Zotero + Better BibTeX** for the bibliographic backbone.
5. **External APIs**: OpenAlex, Semantic Scholar, PubMed, Crossref, Unpaywall, ORCID, ROR for enrichment.
6. **Git** for vault history.
7. **A local REST API or Agent Client Protocol (ACP)** to let Hermes write into Obsidian.
8. **Pandoc** for export.

## Use pre-built skills, don't roll your own

Pre-built skills cover most of the enrichment and ingest work; the agent should use them rather than writing API clients from scratch.

### K-Dense scientific-agent-skills

| Skill | Purpose |
| --- | --- |
| `paper-lookup` | Unified search across 10 databases (PubMed, PMC, bioRxiv, medRxiv, arXiv, OpenAlex, Crossref, Semantic Scholar, CORE, Unpaywall). Includes OpenAlex DOI/ORCID/ROR batch lookups, citation graph traversal, OA PDF discovery, and bibliometrics. |
| `pyzotero` | Read/write Zotero — including writing stable IDs back to the `Extra` field after enrichment. |
| `citation-management` | Crossref DOI resolution and reference normalization. |

### Obsidian skills

Hermes-side skills that operate inside the vault via the Obsidian Local REST API:

- `obsidian-paper-note` — full ingest pipeline (Zotero → PDF → Markdown → vault note).
- Vault read/write skills for note creation, frontmatter updates, and link maintenance.

### Hermes built-in skills

- `llm-wiki` — the umbrella ingest/enrich/triage/draft/lint skill bundle for vault operations. `hermes run llm-wiki ingest --source {citekey}` is the canonical entry point.

### Generic REST Bridge — the escape hatch

A single reusable Hermes skill that wraps any REST endpoint. Used when an API matters once or twice but doesn't justify a dedicated skill yet. Lane-gated to research only via the policy MCP (see [02-profiles.md](../02-profiles.md#lane-override-files)).

| Aspect | Value |
| --- | --- |
| Skill name | `generic-rest-bridge` |
| Inputs | `{base_url, path, method, headers, query, body, timeout, bearer_env}` |
| Output shape | `{ok: bool, status: int, summary: str, data: any, error?: str}` |
| Lane access | Research only |
| Network policy | `external_api_policy: explicit_only` — must be invoked with explicit URL, not via prompt-driven URL synthesis |
| Auth | Reads bearer tokens from environment variables; no secrets in the skill itself |

The rule: if you find yourself calling the same endpoint repeatedly through the bridge, that's the signal to promote it to a dedicated skill (a `<service>-fetch` skill with a narrower contract). The bridge is for the long tail; dedicated skills are for the head.

## Model routing: synthesis on Claude, cheap tasks elsewhere

Not every model call needs the most capable model. The discipline:

| Task class | Examples | Recommended model | Why |
| --- | --- | --- | --- |
| **Synthesis / writing** | `draft`, claim-note generation, wiki page compilation | Claude (the primary synthesis model) | Quality matters most; cost is justified by the irreducibly judgment-laden nature of the work. |
| **Verify / similarity / claim trace** | `cite-check`, `claim-trace`, `similarity-check`, `find-duplicates`, `retraction-check` | Claude (smaller variant acceptable) or cheap model with strong retrieval | Decisions are mostly mechanical; precision matters more than depth. |
| **Code generation** | Coder profile work, delegated to Kilocode / Aider / Claude Code | Claude or Codex via the external coding agent | Best handled by the dedicated coding tool's chosen model, not by Hermes directly. |
| **Bulk / mechanical** | Embedding for `qmd` hybrid search, classification of `_draft_classification` fields, short summaries during enrichment | Cheap model via OpenRouter or similar (e.g., `gemini-flash`, `llama-3.1-8b`) | High volume, low judgment; cost dominates and small models are accurate enough. |

The model selection is configured in each profile's `config.yaml` — not at the workflow layer. The lane-override declares *what skills the profile may run*; the profile's config decides *which model* runs them. This keeps cost discipline operational rather than aspirational:

- The overnight loop's `$1–3/day` budget (see [07-roadmap/future-directions.md](../07-roadmap/future-directions.md)) is achievable *because* embed and classify calls go to a cheap model; routing everything through Claude would push the budget several times higher.
- Cost regressions surface in the [fleet-observability dashboard](../dashboards/fleet-observability.md) at the per-skill level — if `paper-lookup` cost-per-task triples, the routing config drifted (or a cheap model was retired and the call now silently falls back to Claude).

OpenRouter and similar gateways (Kilopass, OpenAI's own model fan-out) are the practical access path to cheap models — they expose a single API surface with many backend models, so the profile config picks the model name rather than negotiating credentials per provider.

## Plugins, apps, and external tools

These are not skills — they are the surrounding ecosystem the agent integrates with.

| Tool | Role |
| --- | --- |
| **Scite** | Citation context (supporting / contrasting / mentioning signals). |
| **Marker** | PDF full-text extraction (no GROBID needed for this workflow). |
| **MarkItDown** | Fallback extractor for non-PDF sources. |
| **MarkDB-Connect** | Obsidian plugin — link Zotero items to vault notes. |
| **qmd** | Hybrid BM25 + vector search over the vault. |
| **Obsidian Local REST API** | Obsidian plugin — exposes the vault to Hermes for read/write. **Distinct from ACP**: REST API is *vault-level* read/write; ACP is *editor-level* agent interface. Complementary — REST API gives Hermes vault access; ACP exposes Hermes to Obsidian / VS Code / Zed agent panes. |
| **Claude API** | The primary synthesis model. |
| **Kilocode or Aider** | Repository-level coding work delegated by the coder profile. |

## Related

- Obsidian plugins (detailed per-plugin configuration): [operations/obsidian-plugins/README.md](../operations/obsidian-plugins/README.md)
- Per-profile skill catalogs: [02-profiles.md](../02-profiles.md)
- Lane-override mechanism: [02-profiles.md lane-override files](../02-profiles.md#lane-override-files)
- Cost-discipline dashboard: [dashboards/fleet-observability.md](../dashboards/fleet-observability.md)
