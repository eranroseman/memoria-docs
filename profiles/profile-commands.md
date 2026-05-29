---
mode: reference
audience: operator
topic: profiles
---

# Profile commands (reference)

The Core commands column in the [Lane permissions matrix](README.md#lane-permissions-matrix) lists the verbs each profile uses; per-profile skill and tool surfaces are documented in the same matrix and in the per-profile design summaries in this folder (`librarian.md`, `mapper.md`, etc.). Beyond the core verbs, this is the full operational command catalog with dry-run defaults and owner profiles. Many of these are also described inline in [workflows/README.md](../workflows/README.md).

| Command | Purpose | Primary profile | Dry-run? |
| --- | --- | --- | --- |
| `ingest` | Create the right note type in the right folder with enrichment. | Librarian | No (creates notes) |
| `find` | Forward / backward citation search or concept-driven search. | Librarian | No (writes candidates) |
| `enrich` | Re-run API enrichment on existing notes. | Librarian | No |
| `classify` | Re-propose `_proposed_classification` when a note still needs review. | Librarian | No |
| `obsidian-paper-note` | Full ingest pipeline including PDF extraction (via Marker). | Librarian | No |
| `export prior-labels` | Export vault papers as ASReview priors for pre-ingest screening. | Librarian | N/A |
| `scope-project` | Map the corpus for a project: cluster density, recency distribution, source diversity, gap analysis. Writes `corpus-map.md` to the project folder. | Mapper | No (writes to project scratch only) |
| `gap-report` | Identify thin-coverage topics adjacent to a project brief. Feeds Verifier's gap-card spawning. | Mapper | No |
| `cluster-map` | Render a density / recency map for an arbitrary topic across the corpus. | Mapper | No |
| `comparative-brief` | When a new source enters the queue, generate a brief comparing it against existing claims — overlap, contradiction, new constructs. Drives the inline `[!brief]` callout. | Mapper | No |
| `socratic-processing` | Question-only conversation about a paper note. No writes. | Socratic | N/A (read-only profile) |
| `lens-reading` | Read a note or cluster through a named theoretical lens. Parameterized by lens name. | Socratic | N/A (read-only profile) |
| `draft` | Search the vault and produce an answer note for human review. | Writer | No |
| `cite-check` | Verify citations in drafts before export — every citekey resolves to a real paper note. | Verifier | Yes (default) |
| `find-duplicates` | Identify semantically similar claim notes for merge review. Retrospective; runs monthly. | Verifier | Yes (always — never auto-merges) |
| `similarity-check` | Point-of-action check: before a new claim note is filed, surface the top 3 most-similar existing notes. Flag at threshold ~0.8; never block. | Verifier | Yes (always — informational, never auto-merges) |
| `retraction-check` | Scan paper notes against Zotero retraction alerts and CrossRef. | Verifier | Yes (default) |
| `lint` | Structural health check across the vault. | Linter | Yes (default) |
| `schema-check` | Verify frontmatter against the authoritative schema. | Linter | Yes |
| `schema-migrate` | Propose schema changes between versions. **Always dry-run first.** | Linter | Yes (always required first) |
| `graph-analyze` | Knowledge graph health: orphans, hubs, clusters, link density. | Linter | Yes |
| `session-log` | Write per-session log file to `00-meta/02-logs/`. | Linter | N/A |

Rule: any command that writes to canonical folders (`30-synthesis/01-claims/`, `30-synthesis/03-moc/`, `50-deliverables/`) or runs a migration must default to dry-run. `schema-migrate` in particular must never be run without reviewing the diff first.

## Related

- [profiles/README.md](README.md) — profile overview and the Lane permissions matrix
- [workflows/README.md](../workflows/README.md) — workflows that orchestrate these commands

<!-- memoria-nav -->

---

[← Previous: Hermes profiles](README.md)

[Next: Librarian — design summary →](librarian.md)
