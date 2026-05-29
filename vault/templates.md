---
mode: reference
audience: operator
topic: vault
---

# Note types and templates (reference)

The 15 note types, their owners, lifecycles, and the templates that produce them. For the conceptual model (folder structure, promotion map, routing rules) see [README.md](README.md); for frontmatter and field-level discipline see [frontmatter-schema.md](frontmatter-schema.md).

## Note types

The 15 note types, consolidated. **Created by** and **Edited by** name a Hermes profile when the agent writes; **Human** means the human is authoritative. **Promoted by** identifies who advances the note through its lifecycle states.

| Note type | Folder | Created by | Edited by | Promoted by | Lifecycle |
| --- | --- | --- | --- | --- | --- |
| `fleeting-note` | `10-inbox/01-fleeting/` | Human | Human | Human (or discard) | `proposed` → `archived` |
| `answer-note` | `10-inbox/02-answers/` | Writer | Writer until review, then human | Human → `claim-note` | `proposed` → `archived` |
| `paper-note` | `20-sources/01-papers/` | Librarian | Librarian (enrichment) + human (synthesis, classification) | Human classification | `proposed` → `current` |
| `item-note` | `20-sources/02-items/` | Librarian | Librarian (enrichment) + human (interpretation) | Human classification | `proposed` → `current` |
| `person-note` | `20-sources/03-entities/01-people/` | Librarian | Librarian (enrichment) + human (relationship fields) | Human classification | `proposed` → `current` |
| `organization-note` | `20-sources/03-entities/02-organizations/` | Librarian | Librarian + human | Human classification | `proposed` → `current` |
| `venue-note` | `20-sources/03-entities/03-venues/` | Librarian | Librarian + human | Human classification | `proposed` → `current` |
| `claim-note` | `30-synthesis/01-claims/` | Human | Human (Writer may suggest links) | Human | `current` (+ `maturity`) |
| `reference-note` | `30-synthesis/02-reference/` | Writer (draft) | Writer + human | Human finalizes | `proposed` → `current` |
| `moc` | `30-synthesis/03-moc/` | Human | Human (Writer may suggest membership) | Human | `current` / `dormant` / `archived` |
| `project-note` | `40-workbench/01-projects/` | Human | Human (Coder may scaffold) | Human | `proposed` → `current` (+ `project_phase`) |
| `code-note` | `40-workbench/01-projects/*/code/` | Coder or human | Coder or human | Human (review gate) | `proposed` → `current` → `archived` |
| `canvas` | `40-workbench/01-projects/*/canvas/` | Human | Human | Human | `proposed` → `current` |
| `draft` | `40-workbench/01-projects/*/drafts/` | Human | Human (Writer assists on request) | Human → `deliverable` on export | `proposed` → `current` (+ `draft_stage`) |
| `deliverable` | `50-deliverables/` | Human (Coder may run export on explicit task) | Terminal — never edited | Terminal | `current` |

Lifecycle states are operational: dashboards query them, promotion gates require them, the Linter validates them.

### Naming convention: `-note` vs bare names

The `-note` suffix is not decorative — it marks a **knowledge node**: the authoritative record of one unit (an idea, source, entity, concept, project, code artifact, or answer) that carries its own content. Of the 15 types, 11 are knowledge nodes and carry the suffix.

The four bare names are deliberately *not* knowledge nodes — they are either a **view** over other notes or an **output** produced from them:

| Bare type | Why no suffix |
| --- | --- |
| `moc` | A **view** — an index/map of other notes. Remove the notes it points at and it is empty. |
| `canvas` | A **view** — a spatial arrangement of other notes (and a `.canvas` JSON file, not markdown). |
| `draft` | A **pipeline output** — a manuscript assembled from claim notes, headed to export. |
| `deliverable` | A **pipeline output** — the exported artifact (often not markdown). |

Two rules govern the suffix:

- **Definition (meaning).** `-note` ⇔ a curated knowledge node. Bare ⇔ a *view* over the graph (`moc`, `canvas`) or an *output* produced from it (`draft`, `deliverable`). This is why `answer-note` keeps the suffix — it is a candidate record entering the inbox → `claim-note` pipeline — while `draft` drops it: it is a document entering the draft → `deliverable` pipeline. The suffix marks *membership of the knowledge graph as a node the human curates*, not mere residence in the vault. (Atomicity is a separate property — only `claim-note` and `fleeting-note` are atomic — and is read off the note type, not the suffix.)
- **Format floor (checkable).** Every `-note` type must be a markdown note in the schema (carries `type` + a lifecycle field). This is one-directional: markdown-ness is *necessary* for `-note` but does not *earn* it — `moc` is a markdown schema note too, yet stays bare because it is a view, not a node.

**Edge case.** `project-note` is the most hub-like knowledge node — it coordinates other work. It keeps the suffix because it is the authoritative record *of a project* (its own scope/status content), not merely a map of other notes. It is the one type where node-vs-view is a judgment call rather than obvious.

### Type-specific behavior

A handful of types have constraints worth calling out.

- **`fleeting-note`** — one idea, one quote, or one task; no polish needed. Promoted or discarded within ~7 days; never lingers.
- **`paper-note`** — `partial` means the Librarian has populated `_proposed_classification` but the human has not promoted fields. `full` means classification is complete. Paper notes are never rewritten as claim notes; they stay tied to their source.
- **`claim-note` is human-only writing.** The Writer profile may suggest links, but the canonical claim text is human-authored. One claim per note. `seedling` = one source; `budding` = multi-source, linked; `evergreen` = stable, ready for `30-synthesis/02-reference/`.
- **`reference-note` requires human finalization.** The Writer drafts; the human signs off. `current` only after review and link consolidation.
- **`code-note`** is the only shared-ownership type. The Coder writes or modifies; the human reviews via the standard review gate.
- **`moc` is curation, not catalogue.** A MOC adds overview, annotated entries, and gaps — not just a list of wikilinks. `dormant` MOCs are kept but not surfaced in dashboards.
- **`deliverable`** is terminal. Once exported, never edited in place — supersede with a new deliverable if changes are needed.

### Jupyter notebooks

Jupyter notebooks (`.ipynb`) are treated as a `code-note` with `format: notebook`. The notebook file itself lives alongside the markdown note in `40-workbench/01-projects/*/code/`. The markdown note carries provenance, purpose, and links; the notebook holds the executable artifact. **Do not** create a separate `notebook-note` type — the discipline is the same (provenance, purpose, motivating literature), only the file format differs.

## Lifecycle

Every note carries one universal field, **`lifecycle`** — its durability phase. (`status` is reserved for board cards; see [board/states.md](../board/states.md).) Types that need finer state within a phase carry a **refinement** field.

| `lifecycle` | Meaning |
| --- | --- |
| `proposed` | created, not yet accepted as durable |
| `current` | accepted, in use, durable |
| `dormant` | set aside / paused, not retired |
| `archived` | retired — superseded, deprecated, discarded |

`archived` is available to every type (any note can be retired), even where the table below doesn't list it as a routine phase.

### Per-type lifecycle range + refinement

| Note type | `lifecycle` range | Refinement (field within lifecycle) |
| --- | --- | --- |
| `fleeting-note` | `proposed` → `archived` (transient — never `current`) | — |
| `answer-note` | `proposed` → `archived` (transient) | — (review tracked on the board card's `review_status`) |
| `paper-note` | `proposed` (pre-classification) → `current` (classified) | `pub_status` (orthogonal — publication state, not lifecycle) |
| `item-note` | `proposed` → `current` | `maintenance_status`, `role_in_stack` (orthogonal) |
| `person-note` / `organization-note` / `venue-note` | `proposed` → `current` | — |
| `claim-note` | `current` from authoring | `maturity`: `seedling` → `budding` → `evergreen` |
| `reference-note` | `proposed` → `current` | — |
| `moc` | `current` / `dormant` / `archived` | — |
| `project-note` | `proposed` → `current` → `dormant`/`archived` | `project_phase`: `planning` / `active` / `paused` / `complete` |
| `code-note` | `proposed` → `current` (active) → `archived` (deprecated) | — |
| `canvas` | `proposed` → `current` (frozen) | — |
| `draft` | `proposed` → `current` (submitted) | `draft_stage`: `outline` / `in-progress` / `submitted` |
| `deliverable` | `current` (final) | — |

The `lifecycle` enum and each refinement field are validated by the Linter's `schema-check` (see [profiles/linter.md](../profiles/linter.md)). New allowed values are added here first, then to the templates. The `lifecycle` value set is deliberately kept **distinct from board-card `status`** — see [frontmatter-schema.md](frontmatter-schema.md#rules) for the full disambiguation.

## Templates

The 15 note templates ship at `00-meta/03-templates/` in the [memoria-vault starter vault](https://github.com/eranroseman/memoria-vault/tree/main/00-meta/03-templates) — one markdown file per note type, read by Obsidian's Templater plugin. They contain the frontmatter shape, body skeleton, and per-type field notes (e.g., the `pub_status` / `full_text_reviewed` semantics for `paper-note`, the maturity progression for `claim-note`, Jupyter handling for `code-note`).

The 15 templates (one per note type):

`fleeting-note.md` · `answer-note.md` · `paper-note.md` · `item-note.md` · `person-note.md` · `organization-note.md` · `venue-note.md` · `claim-note.md` · `moc.md` · `reference-note.md` · `project-note.md` · `code-note.md` · `canvas.md` · `draft.md` · `deliverable.md`

Templates are content shapes, not architectural concepts — they don't have separate design summaries here in memoria-docs. The runtime files in the vault repo are the authoritative spec. The frontmatter rules they encode are governed by [vault/frontmatter-schema.md](frontmatter-schema.md); the Linter's `dashboard-field-drift` detector ([Linter design summary](../profiles/linter.md)) catches Dataview queries that reference fields no template emits.

### Config templates (not note types)

One additional template ships for a *config* artifact rather than a content note type:

| Artifact | Template file | Destination |
| --- | --- | --- |
| `design-system` | [surfaces/design-system.md](../surfaces/design-system.md) | `00-meta/04-reference/design-system.md` |

The design-system file isn't one of the 15 note types — it's a single-instance config artifact that drives the vault's visual style, and the template follows [open-design](https://github.com/nexu-io/open-design)'s portable DESIGN.md format so the same file can drive open-design's render pipeline directly. For what consumes the rendered file, see [README.md](README.md#vault-skeleton-human-facing-notes).

<!-- memoria-nav -->

---

[← Previous: Notes, folders, and linking](README.md)

[Next: Frontmatter schema (reference) →](frontmatter-schema.md)
