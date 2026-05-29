---
mode: explanation
audience: system-designer
topic: decisions
id: 3
title: Answer-draft retention
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-3: Answer-draft retention

## Context

Agent-drafted `answer-note` files land in `10-inbox/02-answers/`. Without a retention policy, the inbox accumulates drafts indefinitely — most never get reviewed, but a few are genuinely useful and shouldn't be lost.

## Decision

Surface unreviewed answer drafts in the weekly dashboard. After a configurable threshold (default 90 days), the Linter flags them for the human to keep, promote, or discard. No auto-archive.

## Consequences

- Drafts remain visible until the human decides their fate.
- The 90-day threshold prevents the inbox from becoming a permanent graveyard.
- Human owns every discard — the agent never silently retires its own work.

## Alternatives considered

**Auto-archive at 90 days** (move to `95-archive/synthesis/` without review): rejected because the most useful drafts are often the ones the human hasn't gotten to yet. Silent archival would hide them at the moment they become most likely to be needed.

**Keep forever**: rejected because the inbox accumulates clutter that erodes the "the inbox is a queue" discipline.

## Related

- **Workflows affected:** [Distill](../workflows/README.md), [Lint](../workflows/README.md)
- **Files affected:** [profiles/linter.md](../profiles/linter.md), [weekly-review.md](../dashboards/weekly-review.md)
- **Resolves / supersedes:** none

<!-- memoria-nav -->

---

[← Previous: ADR-2: Auto-promotion threshold](02-auto-promotion-threshold.md)

[Next: ADR-4: Citekey naming convention →](04-citekey-naming-convention.md)
