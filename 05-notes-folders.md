# 05 — Notes, folders, and linking

The vault is where durable knowledge lives. This document covers:

- The layered folder structure and what each layer means.
- The fifteen canonical note types, their owners, and lifecycles.
- Frontmatter namespace discipline.
- Linking patterns and the expected link topology.

## Folder structure

Folders encode **lifecycle stage**, not subject area. The top-level number tells you where in the capture → describe → think → work → ship progression a note sits.

```text
<vault-root>/                  ← starter vault; operator picks the folder name
├── 00-meta/
│   ├── 01-templates/
│   ├── 02-csl/
│   ├── 03-config/
│   ├── 04-logs/
│   ├── 05-dashboards/
│   ├── 06-schema/
│   ├── 07-skills/
│   └── 08-metrics/
├── 10-inbox/
│   ├── 01-fleeting/
│   ├── 02-synthesis/
│   └── 03-candidates/
├── 20-sources/
│   ├── 01-literature/
│   ├── 02-items/
│   └── 03-entities/
│       ├── 01-people/
│       ├── 02-organizations/
│       └── 03-venues/
├── 30-synthesis/
│   ├── 01-permanent/
│   ├── 02-wiki/
│   └── 03-moc/
├── 40-workbench/
│   ├── 01-projects/
│   ├── 02-drafts/
│   ├── 03-code/
│   └── 04-canvas/
├── 50-deliverables/
│   ├── 01-manuscripts/
│   ├── 02-presentations/
│   └── 03-exports/
├── 90-assets/
├── 95-archive/
│
├── .obsidian/                 ← Obsidian config (auto-hidden)
└── .memoria/                  ← Memoria tooling (dot-prefix: auto-hidden by Obsidian)
    ├── profiles/              ← seven hand-authored Hermes profile directories
    ├── mcp/                   ← policy_mcp.py, tasks_mcp.py, requirements.txt
    └── lane-overrides/        ← per-lane YAML the policy MCP reads at startup
```

The vault is a single repo, opened directly by Obsidian. The numbered top-level folders are operator-visible workspace; `.obsidian/` and `.memoria/` (both dot-prefixed and therefore auto-hidden by Obsidian's vault scanner) hold tooling. `.memoria/profiles/` is the source of truth for the seven Hermes profiles — `install.ps1` copies these verbatim into `~/.hermes/profiles/memoria-<name>/` at install time. See [01-architecture/on-disk-layout.md](01-architecture/on-disk-layout.md) for the full picture, including the install flow and the source-vs-runtime relationship.

### Why this layout

The biggest gain over a flat structure is that **source**, **synthesis**, **workbench**, and **deliverable** become visibly different zones. The agent and human both see at a glance what a folder is *for*: things that describe the world (sources), things that express your thinking (synthesis), things being worked on (workbench), and things that have shipped (deliverables).

The split inside `30-synthesis/` (`permanent/`, `wiki/`, `moc/`) is especially load-bearing because it makes the three kinds of conceptual knowledge distinct: claims, reference pages, and navigation hubs.

### Folder roles and access

Coarse access summary. The authoritative per-profile permissions live in [02-profiles.md](02-profiles.md#lane-permissions-matrix); per-write policy is enforced by the policy MCP via the [lane-override files](reference/policy-mcp.md).

| Folder | Role | Human | Agent |
| --- | --- | --- | --- |
| `00-meta/` | Schema, templates, dashboards, logs, configuration. | Full | Read; write logs and dashboards only. |
| `00-meta/01-templates/` | Note templates. | Full | Read. |
| `00-meta/02-csl/` | Citation style files for Pandoc. | Full | Read. |
| `00-meta/03-config/` | System configuration. | Full | Read. |
| `00-meta/04-logs/` | Session logs and audit trail. | Full | Write (own logs). |
| `00-meta/05-dashboards/` | Dataview dashboard pages. | Full | Read; update under instruction. |
| `00-meta/06-schema/` | Schema documents, AGENTS contracts. | Full | Read. |
| `00-meta/07-skills/` | One `skill-note` per Hermes skill. | Full | Read; write under skill onboarding only. |
| `00-meta/08-metrics/` | `lane-metric` and `skill-metric` notes for fleet observability. | Read | Write (aggregator only). |
| `10-inbox/01-fleeting/` | Raw captures, unprocessed thoughts. | Write | Read / write. |
| `10-inbox/02-synthesis/` | Draft answers awaiting review. | Review | Write. |
| `10-inbox/03-candidates/` | Discovery leads and screening queue. | Review | Write. |
| `20-sources/01-literature/` | One note per citable source (paper, dataset, report, software). | Review | Write. |
| `20-sources/02-items/` | Tools, repos, packages, products, standards. | Review | Write. |
| `20-sources/03-entities/01-people/` | People (authors, advisors, collaborators, developers). | Review | Write. |
| `20-sources/03-entities/02-organizations/` | Labs, universities, companies, funders. | Review | Write. |
| `20-sources/03-entities/03-venues/` | Journals, conferences, workshops. | Review | Write. |
| `30-synthesis/01-permanent/` | Durable claims in your own words. | Write | Read; suggest links only. |
| `30-synthesis/02-wiki/` | Stable reference pages. | Review / edit | Draft and limited updates. |
| `30-synthesis/03-moc/` | Maps of Content; navigation hubs. | Write | Read; suggest only. |
| `40-workbench/01-projects/` | Project coordination, scope, milestones. | Write | Read / write. |
| `40-workbench/02-drafts/` | Manuscripts in progress. | Write | Read only unless asked. |
| `40-workbench/03-code/` | Code artifacts and scripts (including Jupyter notebooks). | Write | Read / write. |
| `40-workbench/04-canvas/` | Argument mapping, spatial planning. | Write | Read only or limited assist. |
| `50-deliverables/` | Final manuscripts, slides, submission-ready exports. | Write | Read / write on explicit export tasks. |
| `90-assets/` | Machine-extracted markdown and non-PDF binary attachments. `90-assets/extracts/<citekey>.md` holds Marker output from ingest; other binaries (figures, datasets, supplementary materials) live alongside as needed. **PDFs do not live here** — they stay in Zotero's storage and are reached via `pdf_uri` on the source-note. | Hidden / managed | Read only. |
| `95-archive/` | Deprecated, superseded, or archived notes. | Read only | Read only. |

## Special files

A small set of `00-meta/` files are not folders and not templates — they are operator-facing singletons that shape how the system runs.

| File | Purpose | Owner |
| --- | --- | --- |
| `00-meta/research-directions.md` | Current research priorities, open questions, synthesis gaps, papers to prioritize. The researcher reads this at session start. | Human (refresh weekly) |
| `00-meta/system-status.md` | Runtime health snapshot: HTTP bridge running, MCPs up, plugin enabled, profiles available. **Distinct from `board-state`**, which tracks work in flight. | Human (occasional refresh) |

`research-directions.md` is the most operationally load-bearing of these: an empty or stale file produces an unfocused researcher. See [04-workflows.md](04-workflows.md) for how it feeds the discover workflow.

## Vault skeleton: operator-facing notes

A freshly-cloned vault ships with a small set of plain-language operator notes in `00-meta/`. These are the *human-facing* counterpart to the technical contracts in `.memoria/profiles/memoria-<name>/SOUL.md` (in the vault) and the design repo's `reference/` directory — they let someone opening the vault for the first time understand what they're looking at without reading the design docs.

| Note | Purpose | Maintained by |
| --- | --- | --- |
| `00-meta/index.md` | Vault landing page. Pinned in sidebar. Links to system status, dashboards, lane views, key files. | Human (rarely changes) |
| `00-meta/getting-started.md` | First-time setup checklist. The 5 steps from clone to first ingest. | Human (rarely changes) |
| `00-meta/system-map.md` | High-level architecture summary in plain language. The vault-resident companion to [01-architecture.md](01-architecture.md). | Human (sync with design changes) |
| `00-meta/agent-roles.md` | Plain-language one-paragraph summary of each Hermes profile. Companion to the SOUL.md contracts at `.memoria/profiles/memoria-<name>/SOUL.md` in the vault. | Human (sync with profile changes) |
| `00-meta/profile-policies.md` | Plain-language summary of who-can-write-where. Companion to the lane-override YAML files and the [Lane permissions matrix](02-profiles.md#lane-permissions-matrix). | Human (sync with lane-override changes) |
| `00-meta/schema-reference.md` | Canonical list of every frontmatter field used in the vault, with type and allowed values. The source of truth that templates and the linter point at. | Human + linter (linter flags drift) |
| `00-meta/dataview-cheatsheet.md` | Reference patterns for dashboard authors — TABLE / LIST / TASK / FROM / WHERE / SORT / FLATTEN / LIMIT examples. | Human (rarely changes) |
| `00-meta/performance-checklist.md` | Dashboard performance discipline (see [06-surfaces.md](06-surfaces/persistent.md#performance-discipline)). | Human (rarely changes) |
| `00-meta/safe-mode.md` | The three core workflows (ingest, review, export) with minimal commands and fallbacks when something is broken. Open this when Hermes, the ACP bridge, or the watcher is down. Pairs with [operations/failure-modes.md](operations/failure-modes.md) for the Detect/Fix/Verify recipes. | Human (rarely changes) |
| `00-meta/obsidian-config.md` | Plain-language summary of which Obsidian community plugins Memoria uses and the load-bearing settings the operator should not change. Companion to [operations/obsidian-plugins/README.md](operations/obsidian-plugins/README.md). | Human (sync with plugin changes) |
| `00-meta/06-schema/design-system.md` | Canonical visual-style source for the vault — palette, typography, spacing, layout, components, motion, voice, brand, anti-patterns. Format follows [open-design](https://github.com/nexu-io/open-design)'s 9-section DESIGN.md schema so the same file can drive open-design's render pipeline. Read by CSS-snippet generators, by Pandoc export configs, and by open-design when rendering deliverables. Templated by [reference/design-system.md](reference/design-system.md). | Human (edits define the brand); design-system schema versioned independently |

The design folder is the *engineering* spec — it describes how to build and reason about the system. The vault skeleton is what an *operator* needs in front of them while using the vault day-to-day. The skeleton notes are intentionally short and plain-language; if a section needs architectural detail, it links to the relevant `memoria-docs/` document.

### Drift discipline

When the design changes — a new profile added, a lane-override rule updated, a schema field introduced — the corresponding skeleton note must be updated. The linter's structural-drift check (see [profiles/linter.md](profiles/linter.md)) flags skeleton notes whose `updated_at` is older than the corresponding design file. Treat skeleton drift the same way you'd treat code-doc drift: pay it down promptly.

## Note types

The fifteen canonical note types, consolidated. **Created by** and **Edited by** name a Hermes profile when the agent writes; **Human** means the human is authoritative. **Promoted by** identifies who advances the note through its lifecycle states.

| Note type | Folder | Created by | Edited by | Promoted by | Lifecycle states |
| --- | --- | --- | --- | --- | --- |
| `fleeting-note` | `10-inbox/01-fleeting/` | Human | Human | Human (or discard) | `unprocessed`, `promoted`, `discarded` |
| `synthesis-note` | `10-inbox/02-synthesis/` | Writer | Writer until review, then human | Human → `claim-note` | `unreviewed`, `in-review`, `approved`, `rejected` |
| `source-note` | `20-sources/01-literature/` | Researcher | Researcher (enrichment) + human (synthesis, triage) | Human triage | `partial`, `full` |
| `item-note` | `20-sources/02-items/` | Researcher | Researcher (enrichment) + human (interpretation) | Human triage | `partial`, `full` |
| `person-note` | `20-sources/03-entities/01-people/` | Researcher | Researcher (enrichment) + human (relationship fields) | Human triage | `partial`, `full` |
| `org-note` | `20-sources/03-entities/02-organizations/` | Researcher | Researcher + human | Human triage | `partial`, `full` |
| `venue-note` | `20-sources/03-entities/03-venues/` | Researcher | Researcher + human | Human triage | `partial`, `full` |
| `claim-note` | `30-synthesis/01-permanent/` | Human | Human (writer may suggest links) | Human | `seedling`, `budding`, `evergreen` |
| `reference-note` | `30-synthesis/02-wiki/` | Writer (draft) | Writer + human | Human finalizes | `draft`, `canonical` |
| `moc` | `30-synthesis/03-moc/` | Human | Human (writer may suggest membership) | Human | `active`, `dormant`, `archived` |
| `project-note` | `40-workbench/01-projects/` | Human | Human (coder may scaffold) | Human | `planning`, `active`, `paused`, `complete`, `archived` |
| `code-note` | `40-workbench/03-code/` | Coder or human | Coder or human | Human (review gate) | `draft`, `active`, `deprecated` |
| `canvas` | `40-workbench/04-canvas/` | Human | Human | Human | `draft`, `frozen` |
| `draft` | `40-workbench/02-drafts/` | Human | Human (writer assists on request) | Human → `deliverable` on export | `outline`, `in-progress`, `submitted` |
| `deliverable` | `50-deliverables/` | Human (coder may run export on explicit task) | Terminal — never edited | Terminal | `final` |

Lifecycle states are operational: dashboards query them, promotion gates require them, the linter validates them.

### Type-specific behavior

A handful of types have load-bearing constraints worth calling out.

- **`fleeting-note`** — one idea, one quote, or one task; no polish needed. Promoted or discarded within ~7 days; never lingers.
- **`source-note`** — `partial` means the researcher has populated `_draft_classification` but the human has not promoted fields. `full` means triage is complete. Source notes are never rewritten as claim notes; they stay tied to their source.
- **`claim-note` is human-only writing.** The writer profile may suggest links, but the canonical claim text is human-authored. One claim per note. `seedling` = one source; `budding` = multi-source, linked; `evergreen` = stable, ready for `30-synthesis/02-wiki/`.
- **`reference-note` requires human finalization.** The writer drafts; the human signs off. `canonical` only after review and link consolidation.
- **`code-note`** is the only shared-ownership type. The coder writes or modifies; the human reviews via the standard review gate.
- **`moc` is curation, not catalogue.** A bare list of wikilinks is not a MOC. MOCs add overview, annotated entries, and gaps. `dormant` MOCs are kept but not surfaced in dashboards.
- **`deliverable`** is terminal. Once exported, never edited in place — supersede with a new deliverable if changes are needed.

### Jupyter notebooks

Jupyter notebooks (`.ipynb`) are treated as a `code-note` with `format: notebook`. The notebook file itself lives alongside the markdown note in `40-workbench/03-code/`. The markdown note carries provenance, purpose, and links; the notebook holds the executable artifact. **Do not** create a separate `notebook-note` type — the discipline is the same (provenance, purpose, motivating literature), only the file format differs.

## Common pitfalls

These failure modes recur. Treat them as anti-patterns to actively avoid.

### Capture and triage

- **Unpinned citekeys.** If you change Zotero metadata before pinning the BBT key, the key regenerates and every wikilink pointing to the old key breaks silently. **Pin all keys immediately after import.**
- **Promoting `_draft_classification` without checking the vocabulary.** If the schema says `receptivity-detection` and you promote `topic: [receptivity]`, queries miss the note. Always check vocabulary at triage.
- **Writing claim notes before triaging the source note.** The `_draft_classification` often surfaces project connections that should appear in the claim note. Triage first.
- **Letting `10-inbox/` accumulate without review.** The inbox is a queue, not storage. Notes older than 7 days are either worth promoting or worth discarding. An inbox that grows is a system capturing without synthesizing — the most common failure mode.

### Synthesis quality

- **Summary notes masquerading as synthesis.** A source note that lists bullet points is a summary, not synthesis. Synthesis means connecting what the paper says to what you already know — if there are no wikilinks in the Summary section, no synthesis has happened.
- **Too many claims in one claim note.** If the title contains "and" doing real work, split the note. One claim, one note.
- **Treating agent synthesis as verified content.** A `synthesis-note` is a proposal. The agent may cite a paper for a claim that paper doesn't actually make. Always verify citekeys before promoting.

### Promotion and structure

- **Using `30-synthesis/02-wiki/` for work-in-progress.** Reference notes should be stable. If a page changes substantially every week, it's a claim note with `maturity: budding`, not a reference note. Move it back.
- **MOC-as-folder-dump.** A MOC that's just a list of every note on a topic is a folder. MOCs add context and annotation. A bare list of wikilinks is not a MOC.
- **Orphan claim notes.** A claim note with no incoming links has not made it into the knowledge graph. The weekly orphan query catches these — act on them.

### Operational hazards

- **Running `schema-migrate` without reviewing the diff.** Running it without `--dry-run` first can silently alter frontmatter across hundreds of notes. **Always dry-run first.**
- **Auto-promotion of `_draft_classification`.** The agent should never promote draft fields without explicit triage. If you find yourself wanting to, write a better dashboard query that surfaces candidates for batch review instead.

## Routing rules

- Route each note to the folder that matches its **type**, not its **topic**.
- If the type is ambiguous, ask before creating or moving the note.
- Do not store a `source-note` under `40-workbench/`, or a `claim-note` in the inbox.
- Do not use `30-synthesis/02-wiki/` for unstable work-in-progress.
- Do not use `30-synthesis/01-permanent/` for source summaries.
- Do not use `40-workbench/02-drafts/` as a general notes folder.
- If a note could fit more than one type, choose the **most specific epistemic role**.
- If a note's purpose changes substantively, create a new note rather than mutating the old one. Link the new note to the old for provenance.

## Promotion map

The lifecycle moves left-to-right through the folder numbers. The arrows below show legal promotion paths — what type a note becomes when promoted.

```text
fleeting-note ──┬──► synthesis-note ──► claim-note ──► reference-note
                │         (writer)        (human)         (writer + human)
                ├──► source-note (triaged)
                │         (researcher → human)
                └──► (discarded)

claim-note ────► moc membership (via frontmatter moc:)
claim-note ────► draft section (cited in body)
draft ─────────► deliverable (on export)
canvas ────────► draft (informs structure; canvas becomes `frozen`)
code-note ─────► project-note (linked from project)
```

Rules that constrain the map:

- `fleeting-note` is reviewed and either promoted or discarded — it does not linger.
- `synthesis-note` → `claim-note` only after human review.
- `claim-note` → `reference-note` only when the claim is `evergreen` and sufficiently cross-linked.
- `source-note` never becomes a `claim-note`. A source describes what the paper says; a claim is what you think.
- Archived notes remain in place for historical traceability; never delete a note with provenance value. Only humans move notes to `95-archive/`.

## Action policy

The note-types table above answers *who creates and edits*. This section adds *delete and move*:

- **Delete** is universally human-only and discouraged across the board. Prefer archive for anything with provenance value.
- **Move into a canonical folder** (`30-synthesis/01-permanent/`, `30-synthesis/02-wiki/`, `30-synthesis/03-moc/`, `50-deliverables/`) is human-only — these are review-gated.
- **Move to `95-archive/`** is human-only. The agent never archives.
- **Move within working zones** (e.g., `10-inbox/` → `20-sources/` after triage) may be agent-initiated when the type and state warrant it.

## Frontmatter

Frontmatter discipline has two concerns:

- **Field shape** — what fields exist, what values they take, who owns them. The detailed reference (field categorization, controlled vocabularies, Properties UI vs YAML guidance) lives in [reference/schema.md](reference/schema.md). The vault-resident operator companion is `00-meta/schema-reference.md` (see [Vault skeleton](#vault-skeleton-operator-facing-notes)).
- **Namespace ownership** — which namespace a field belongs to and who is allowed to write it. Covered next.

Every note has a `type` field (one of the 15 canonical names) and a single lifecycle field whose name depends on the note type (`triage_status` for sources/items/entities, `maturity` for claim-notes, `status` for everything else). `status` on a note is *not* the same field as `status` on a board card; they share a name but apply to different objects with different value sets.

### Frontmatter namespace discipline

Frontmatter has three namespaces. Each has a different owner.

| Namespace | Owner | Example fields | Rule |
| --- | --- | --- | --- |
| **Main YAML** | Human (authoritative) | `title`, `type`, `topic`, `methods`, `triage_status`, `maturity` | Human-set values are canonical; the agent must never silently rewrite. |
| **`_draft_classification`** | Agent (proposal only) | proposed `topic`, `methods`, `study_design` | Agent populates; human reviews; on triage, selected fields are promoted to main YAML and the draft block is removed. |
| **`_enrichment`** | Agent (maintained) | API-derived fields: citation counts, abstract, venue. The `enriched_date` field sits at the top level (not inside `_enrichment`) because dashboards and the linter's stale-enrichment check query it directly. | Agent refreshes on schedule; values are mutable; never overwrite human edits to corresponding main fields. |

Rules:

- Stable identifiers (DOI, OpenAlex ID, ORCID, ROR) may be promoted from `_enrichment` to main YAML once verified.
- Derived metrics (citation counts, OA status) stay in `_enrichment` — they drift over time.
- The agent must never overwrite human-set frontmatter fields, even if API data contradicts.
- Zotero is the canonical store for bibliographic fields (citekey, DOI, title, authors). The source note references these for navigation but does not duplicate them; Zotero wins on disagreement, the vault wins for synthesis and links.

## Linking patterns

Linking is the cross-cutting discipline that turns the vault into a graph rather than a flat folder hierarchy. Two cardinal rules anchor the practice: **every `claim-note` traces to at least one `source-note` citekey**, and **provenance direction is preserved** — claims point to evidence, never the reverse.

**Full reference** — five link types (`citekey-link`, `concept-link`, `moc-link`, `entity-link`, `agent-cross-link`), the full rule set (including indirect co-authorship and the orphan-rescue rule), the expected cross-link graph by note type, MOC creation thresholds (topic MOC at ≥15–20 notes, domain MOC at ≥3 topic MOCs, child MOC split at >20 claim notes, etc.), and slug collision resolution patterns — live in [reference/linking-patterns.md](reference/linking-patterns.md).

## Templates

The 15 note templates ship at `00-meta/01-templates/` in the [memoria-vault starter vault](https://github.com/eranroseman/memoria-vault/tree/main/00-meta/01-templates) — one markdown file per note type, read by Obsidian's Templater plugin. They contain the frontmatter shape, body skeleton, and per-type field notes (e.g., the `pub_status` / `full_text_reviewed` semantics for `source-note`, the maturity progression for `claim-note`, Jupyter handling for `code-note`).

The 15 templates (one per canonical note type):

`fleeting-note.md` · `synthesis-note.md` · `source-note.md` · `item-note.md` · `person-note.md` · `org-note.md` · `venue-note.md` · `claim-note.md` · `moc.md` · `reference-note.md` · `project-note.md` · `code-note.md` · `canvas.md` · `draft.md` · `deliverable.md`

Templates are content shapes, not architectural concepts — they don't have separate design summaries here in memoria-docs. The runtime files in the vault repo are the authoritative spec. The frontmatter rules they encode are governed by [reference/schema.md](reference/schema.md); the linter's M4 dashboard-field-drift detector ([Linter design summary](profiles/linter.md)) catches Dataview queries that reference fields no template emits.

### Config templates (not note types)

One additional template ships for a *config* artifact rather than a content note type:

| Artifact | Template file | Destination |
| --- | --- | --- |
| `design-system` | [reference/design-system.md](reference/design-system.md) | `00-meta/06-schema/design-system.md` |

The design-system file isn't one of the canonical fifteen note types — it's a single-instance config artifact that drives the vault's visual style (read by CSS-snippet generators, Pandoc export configs, and [open-design](https://github.com/nexu-io/open-design) when rendering deliverables). The template follows open-design's portable DESIGN.md format so the same file can drive open-design's render pipeline directly.

## Next

- For surfaces (dashboards, workspaces, inline callouts) that query these notes and folders: [06-surfaces.md](06-surfaces.md).
- For the workflows that produce and promote them: [04-workflows.md](04-workflows.md).
- For the profiles that can read or write each folder: [02-profiles.md](02-profiles.md).
