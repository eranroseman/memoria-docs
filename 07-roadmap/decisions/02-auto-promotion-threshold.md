---
id: 2
title: Auto-promotion threshold
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-2: Auto-promotion threshold

## Context

At what point does a `claim-note` get flagged for promotion to `reference-note`? Three plausible triggers exist: manual operator flagging, a `maturity` field crossing a threshold, or a link-density heuristic (e.g., 5+ inlinks).

## Decision

Manual flagging via `maturity: evergreen`, surfaced by the [weekly dashboard](../../dashboards/weekly-dashboard.md). No automatic promotion to the wiki layer.

## Consequences

- Slower throughput — the operator must explicitly mark notes evergreen.
- No false-positive promotions clogging the canonical `30-synthesis/02-wiki/` layer.
- The weekly ritual carries the promotion decision; promotion never happens silently.

## Alternatives considered

A maturity-driven auto-promotion (set `maturity: evergreen` and the linter auto-moves the note to `30-synthesis/02-wiki/`) would speed throughput. Rejected because the move from claim layer to wiki is semantically meaningful — the operator should pause to confirm.

A link-density heuristic was considered and rejected: it confuses "well-cited" with "stable enough for the wiki layer." Some claim notes accumulate inlinks because they're contested, not because they're evergreen.

## Related

- **Workflows affected:** [#6 Promote to wiki](../../04-workflows.md)
- **Files affected:** [weekly-dashboard.md](../../dashboards/weekly-dashboard.md)
- **Resolves / supersedes:** none
