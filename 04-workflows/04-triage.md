# Workflow 4: Triage

**Group.** Upstream
**Goal.** Turn provisional agent output into canonical metadata and a reviewed source note.

## Steps

1. Human opens the source note.
2. Reviews `_draft_classification`.
3. Promotes selected fields into the main YAML.
4. Deletes the draft block.
5. Sets `triage_status: full` and `triage_completed`.
6. Fills the summary sections.

## Owners

Hermes proposes classification. Human owns all promotion decisions.

## Example

`mamykina2010sense.md` is at `triage_status: partial` with `_draft_classification: { topic: [receptivity-detection], methods: [field-study] }` → human opens the note, agrees with `topic`, refines `methods: [field-study, qualitative-interview]` for accuracy → promotes both fields into the main YAML and deletes the `_draft_classification` block → sets `triage_status: full` and `triage_completed: 2026-05-25` → writes 2–3 sentences in the Key findings section. The note is now canonical for queries and dashboards.

## Related

- **Previous workflow:** [#2 Ingest](02-ingest.md)
- **Next workflow:** [#14 Process](14-process.md)
- **Profile:** [profiles/researcher.md](../profiles/researcher.md)
- **Classifier confidence:** [ADR-11 confidence scoring on `_draft_classification`](../07-roadmap/decisions/README.md)
