---
mode: how-to
audience: operator
topic: workflows
---

# Archive

**Group.** Maintenance
**Goal.** Preserve superseded material instead of deleting it.

## Steps

1. Human marks a note `deprecated`, `superseded`, or `archived`.
2. Adds `superseded_by` if needed.
3. Moves the file to `95-archive/`.
4. The note drops out of active queries but remains in Git history.

## Owners

Human only. Hermes never autonomously archives.

## Related

- **Anti-pattern:** treating archive as delete. Archived notes are preserved for traceability — see [Anti-patterns in workflows/README.md](../README.md#anti-patterns).

<!-- memoria-nav -->

---

[← Previous: Promote](promote.md)

[Next: Assess →](../downstream/assess.md)
