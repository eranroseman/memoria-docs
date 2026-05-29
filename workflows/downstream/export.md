---
mode: how-to
audience: operator
topic: workflows
---

# Export

**Group.** Downstream (stage workflow, added 2026-05)
**Goal.** Run Pandoc to turn a verified draft into the final, frozen deliverable.

## Pipeline position

Terminal downstream stage; runs after [Revise](revise.md), once the verify→revise loop has closed. Produces the deliverable in `50-deliverables/`.

## Precondition

The verify→revise loop ([Verify](verify.md) → [Revise](revise.md)) has closed — the draft is verify-clean, or its remaining gaps are explicitly accepted-soft. Export is blocked while a `verify` card sits in a non-clean state.

## Steps

1. The project card moves to `export` once [Revise](revise.md) closes (or the human advances a verify-clean draft).
2. Run Pandoc against the draft, resolving citekeys against `.memoria/library.bib` with the project's CSL style.
3. The output lands in `50-deliverables/` as the deliverable — terminal and frozen.
4. If the deliverable later needs changes, supersede it with a new draft → export cycle; never edit the exported artifact in place.

## Owners

Human owns the decision to ship; the Coder profile runs the Pandoc pipeline mechanics. Export only reads the draft and writes to `50-deliverables/` — no canonical-knowledge writes.

## Command

```bash
pandoc 40-workbench/01-projects/<project>/drafts/{chapter}.md --citeproc \
  --bibliography .memoria/library.bib \
  --csl .memoria/csl/apa.csl \
  -o 50-deliverables/01-manuscripts/{chapter}.docx
```

## Card lifecycle

`export` (opens when [Revise](revise.md) closes) → `active` (Coder runs Pandoc) → `done` with the deliverable written to `50-deliverables/`. The card cannot enter `active` while the upstream `verify` card is in a non-clean state.

## Why export is its own stage

Export is mechanical, separately triggered (Pandoc, Coder-run), and gated on the verify→revise loop closing — distinct in owner and trigger from the human-led writing it follows. Giving it its own workflow makes "draft written but not yet exported" a visible board state instead of a step buried inside the drafting umbrella.

## Related

- **Previous workflow:** [Revise](revise.md)
- **Umbrella workflow:** [Write](write.md)
- **Profile:** [profiles/coder.md](../../profiles/coder.md) — runs the Pandoc pipeline
- **Output:** the `deliverable` note type — see [vault/templates.md](../../vault/templates.md)

<!-- memoria-nav -->

---

[← Previous: Revise](revise.md)

[Next: Query →](query.md)
