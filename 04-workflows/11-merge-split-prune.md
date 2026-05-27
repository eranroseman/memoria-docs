# Workflow 11: Merge, split, prune

**Group.** Maintenance
**Goal.** Keep claim notes atomic and remove duplication without losing provenance.

## Steps

1. Hermes finds duplicate or similar notes.
2. Human reviews candidate pairs.
3. Human merges, splits, or archives.
4. Backlinks and MOCs are updated.

## Owners

Hermes identifies. Human decides every change.

## Command

```bash
hermes run find-duplicates --folder 30-synthesis/01-permanent --threshold 0.85
```

## Related

- **Profile:** [profiles/verifier.md](../profiles/verifier.md) (runs `similarity-check` and `find-duplicates`)
- **Pre-filing similarity check** also fires during [#5 Synthesize](05-synthesize.md#pre-filing-similarity-check) as a point-of-action duplicate guard.
