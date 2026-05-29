---
mode: how-to
audience: operator
topic: workflows
---

# Classify

**Group.** Upstream
**Goal.** Turn provisional agent output into canonical metadata and a reviewed paper note.

## Steps

1. Human opens the paper note.
2. Reviews `_proposed_classification`.
3. Promotes selected fields into the main YAML.
4. Deletes the proposed block.
5. Sets `lifecycle: current` and `triage_completed`.
6. Fills the summary sections.

## Owners

Hermes proposes classification. Human owns all promotion decisions.

## Example

`mamykina2010sense.md` is at `lifecycle: proposed` with `_proposed_classification: { topic: [receptivity-detection], methods: [field-study] }` → human opens the note, agrees with `topic`, refines `methods: [field-study, qualitative-interview]` for accuracy → promotes both fields into the main YAML and deletes the `_proposed_classification` block → sets `lifecycle: current` and `triage_completed: 2026-05-25` → writes 2–3 sentences in the Key findings section. The note is now canonical for queries and dashboards.

## Related

- **Previous workflow:** [Ingest](ingest.md)
- **Next workflow:** [Discuss](discuss.md)
- **Profile:** [profiles/librarian.md](../../profiles/librarian.md)
- **Classifier confidence:** [ADR-11 confidence scoring on `_proposed_classification`](../../decisions/README.md)

<!-- memoria-nav -->

---

[← Previous: Ingest](ingest.md)

[Next: Discuss →](discuss.md)
