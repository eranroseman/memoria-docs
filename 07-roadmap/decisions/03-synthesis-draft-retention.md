---
id: 3
title: Synthesis-draft retention
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-3: Synthesis-draft retention

## Context

Agent-drafted `synthesis-note` files land in `10-inbox/02-synthesis/`. Without a retention policy, the inbox accumulates drafts indefinitely — most never get reviewed, but a few are genuinely useful and shouldn't be lost.

## Decision

Surface unreviewed synthesis drafts in the weekly dashboard. After a configurable threshold (default 90 days), the linter flags them for the operator to keep, promote, or discard. No auto-archive.

## Consequences

- Drafts remain visible until the operator decides their fate.
- The 90-day threshold prevents the inbox from becoming a permanent graveyard.
- Operator owns every discard — the agent never silently retires its own work.

## Alternatives considered

**Auto-archive at 90 days** (move to `95-archive/synthesis/` without review): rejected because the most useful drafts are often the ones the operator hasn't gotten to yet. Silent archival would hide them at the moment they become most likely to be needed.

**Keep forever**: rejected because the inbox accumulates clutter that erodes the "the inbox is a queue" discipline.

## Related

- **Workflows affected:** [#5 Synthesize](../../04-workflows.md), [#10 Maintenance and linting](../../04-workflows.md)
- **Files affected:** [profiles/linter.md](../../profiles/linter.md), [weekly-dashboard.md](../../dashboards/weekly-dashboard.md)
- **Resolves / supersedes:** none
