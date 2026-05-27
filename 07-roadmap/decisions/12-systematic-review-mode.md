---
id: 12
title: Systematic-review mode
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-12: Systematic-review mode

## Context

Adding a `review_mode: systematic-review` flag plus PRISMA-style fields (inclusion/exclusion criteria, vote traces, multi-reviewer agreement) would support formal scoping reviews and systematic reviews — domains with strict reporting requirements.

## Decision

**Defer.** Adopt only when actively running a systematic review. Build the minimal flag and fields on demand; don't add unused schema.

## Consequences

- Casual reading and synthesis stay free of systematic-review overhead.
- When a systematic review starts, the schema work is non-trivial but localized to that project.
- Per-project adoption means the schema additions never become baseline cost for all source-notes.

## Alternatives considered

**Always-on PRISMA schema**: rejected — most source-notes wouldn't carry meaningful values, and dashboards would have to filter the empty fields constantly.

**Build now in case it's needed**: rejected — premature schema is harder to remove than to add.

## Related

- **Related decisions:** [ADR-18 evidence quality fields](18-evidence-quality-fields.md), [ADR-19 pre-ingest screening](19-pre-ingest-screening.md), [ADR-20 dual-rater workflow](20-dual-rater-workflow.md) — same domain, similar adopt-on-demand pattern.
- **Files affected:** [05-notes-folders.md](../../05-notes-folders.md), [templates/source-note.md](../../templates/source-note.md)
- **Resolves / supersedes:** none
