# Schema reference

The canonical reference for frontmatter discipline: how fields are categorized, which fields use controlled vocabularies, and how the Properties UI vs raw YAML choice is made. The summary lives in [05-notes-folders.md](../05-notes-folders.md#frontmatter); this document is the reference.

The vault ships an operator-facing companion at `00-meta/schema-reference.md` (see the [vault skeleton](../05-notes-folders.md#vault-skeleton-operator-facing-notes)). That note is the in-vault version templates and the linter point at; this document is the design source it's generated from.

## Frontmatter field categorization

Frontmatter fields split into four categories by *what they're for*. The split keeps the core query surface small — every note carries the global and time fields, but only the notes that need them carry domain fields.

| Category | What it's for | Example fields | Present on |
| --- | --- | --- | --- |
| **Global** | Identity. The minimal fields every note carries. | `type`, `schema_version` | Every note |
| **Lifecycle** | Where the note sits in its own state machine. The exact field name varies by note type. | `status`, `triage_status`, `maturity`, `pub_status` | Every note (one type-specific field) |
| **Time** | When the note was created, last touched, or is due. | `created`, `updated`, `enriched_date`, `triage_completed`, `promoted_date` | Every note carries `created` and `updated` (source-note uses `added` instead of `created`); the rest are type-specific |
| **Domain** | Type-specific data. The fields one kind of note uses but others don't. | `citekey`, `doi`, `authors`, `methods`, `topic`, `sources`, `moc`, `projects`, `zotero_uri`, `pdf_uri`, `extract_path` | Only the note types that need them |

## Rules

- **`type` is universal and load-bearing.** Every note has a `type` field whose value is one of the 15 canonical names. Missing or unknown `type` is a schema-hygiene flag for the linter.
- **The lifecycle field is type-specific.** Sources, items, and entities use `triage_status` (`partial` / `full`). Claim notes use `maturity` (`seedling` / `budding` / `evergreen`). Source notes additionally carry `pub_status` for publication state. Everything else uses a plain `status` whose allowed values depend on the note type — see [Lifecycle states by note type](../05-notes-folders.md#lifecycle-states-by-note-type).
- **Time fields are mostly common.** `created` and `updated` are universal except that `source-note` historically uses `added` instead of `created`; treat the two as synonymous for source notes.
- **Domain fields are scoped to their type.** Don't pad every note with null domain fields. A `claim-note` should not carry a `doi` field just to keep the schema "uniform."
- **`status` on a note vs `status` on a card are different fields.** The note's `status` tracks the note's lifecycle (e.g., `draft` / `active` / `archived` for a code-note). The card's `status` tracks board state (e.g., `pending` / `ready` / `active` / `awaiting-review` for a Kanban card). They share a name but live on different objects with different value sets — see [03-board.md](../03-board.md) for card states. The two value sets are deliberately disjoint (no `draft` in card states, no `pending` in note statuses) so a value alone tells you which object it belongs to.
- **Once the design is in use, any frontmatter change to a template bumps that template's `schema_version`.** Adding a field, removing a field, renaming a field, changing a field's value space — all of these are schema changes that require a version bump *if there are existing notes in operator vaults that would lag behind*. New notes get the new version; existing notes stay on the old version until migrated. The linter's schema-version-mismatch check ([profiles/linter.md](../profiles/linter.md)) surfaces notes still on older versions; `schema-migrate --dry-run` proposes the migration. This rule is what turns per-field migration debt into a single rollup signal — "127 notes still on v1" rather than "127 notes missing field X, 89 missing field Y, ..." per field. **Bumping is per-template, not global** — only the template whose schema changed bumps. Source-note and code-note can be on different versions independently. **Pre-first-deployment, the design is fluid; the baseline is `schema_version: 1` for every template and the bump discipline activates the first time a template ships into a vault that holds real notes.**

## Controlled vocabularies

For controlled-vocabulary fields, the allowed values are:

| Field | Where it lives | Allowed values |
| --- | --- | --- |
| `status` (on a card) | Board card | `pending`, `ready`, `active`, `blocked-on-human`, `awaiting-review`, `rejected`, `retry-needed`, `approved`, `done` |
| `status` (on a note) | Note frontmatter | Type-specific — see [Lifecycle states by note type](../05-notes-folders.md#lifecycle-states-by-note-type) |
| `triage_status` | source-note, item-note, entity-* | `partial`, `full` |
| `maturity` | claim-note | `seedling`, `budding`, `evergreen` |
| `pub_status` | source-note | `active`, `preprint`, `retracted`, `deprecated`, `expression-of-concern` |
| `maintenance_status` | item-note | `active`, `deprecated`, `archived`, `unmaintained` |
| `role_in_stack` | item-note | `primary-tool`, `dependency`, `alternative`, `reference-only` |
| `review_state` | Board card | `unreviewed`, `requested`, `in-review`, `approved`, `rejected` |

The linter's `schema-check` (see [linter.md](../profiles/linter.md)) validates frontmatter against this reference. **New allowed values go through this reference first, not the other way around** — a template or query that uses a value not listed here is the bug, not the reference.

## Reach-into-source fields (source-note)

The `source-note` frontmatter carries three deliberate hooks into the underlying paper. Each gives a different reach with different access semantics:

| Field | Form | What it opens | When to use |
| --- | --- | --- | --- |
| `zotero_uri` | `zotero://select/items/<key>` | Zotero item record (metadata, attachments list, tags) | Managing citation metadata, retraction status, attachments |
| `pdf_uri` | `zotero://open-pdf/library/items/<key>` | PDF directly in Zotero's reader | Reading or annotating the paper |
| `extract_path` | Vault-relative path (e.g., `90-assets/extracts/mamykina2010sense.md`) | Marker-extracted markdown — the in-vault, searchable representation | Grep, quoting in claim notes, feeding text to a model |

All three are populated by the Researcher profile during ingest (workflow #2 in [04-workflows/02-ingest.md](../04-workflows/02-ingest.md)) — none are operator-typed. The PDF itself lives in Zotero's storage, not in the vault; see [04-workflows/01-zotero-capture.md](../04-workflows/01-zotero-capture.md) for the canonical-store rationale.

## Properties UI vs YAML frontmatter

Obsidian's Properties UI is a UI layer over the underlying YAML. Same storage, different editor. Use them deliberately:

- **Properties UI** for ad-hoc human edits — single-note changes, quick field updates. Lower syntax-error risk.
- **YAML frontmatter** for system notes, dashboards, registry files, and any Templater-generated note. Better Git diffs, deterministic for automation, full formatting control.

The rule: if the field is being set by code or by a template, treat the YAML as authoritative. If it's being set by a person reading the note, the Properties UI is friendlier.

## Related design documents

- [05-notes-folders.md](../05-notes-folders.md) — folder structure, note types, lifecycle states, namespace discipline (who writes what)
- [linter.md](../profiles/linter.md) — schema-check lint rules that enforce this reference
- [05-notes-folders.md#vault-skeleton-operator-facing-notes](../05-notes-folders.md#vault-skeleton-operator-facing-notes) — the operator-facing companion in `00-meta/schema-reference.md`
