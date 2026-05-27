---
id: 20
title: Dual-rater workflow for inter-rater reliability
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-20: Dual-rater workflow for inter-rater reliability

## Context

Formal scoping reviews and systematic reviews often require inter-rater reliability calculations (Cohen's kappa) on inclusion/exclusion decisions. Adding `rater_1`, `rater_2`, `rater_agreement` fields would support this.

## Decision

**Activate only when the chapter or paper requires it.** Resolve disagreements in a reconciliation session and set `rater_agreement: resolved`.

## Consequences

- Most reading doesn't carry rater-disagreement overhead.
- Systematic reviews have the fields they need.
- Meaningless without a second human rater — adoption requires a collaborator, not just a flag.

## Alternatives considered

**Single-rater always**: works for the default solo workflow but fails the reporting bar for formal reviews. Defensible only outside systematic-review contexts.

**Hermes as the second rater**: rejected explicitly — agent judgment doesn't satisfy inter-rater reliability protocols, which exist precisely to triangulate *human* disagreement.

## Related

- **Related decisions:** [ADR-12 systematic-review mode](12-systematic-review-mode.md), [ADR-19 pre-ingest screening](19-pre-ingest-screening.md).
- **Files affected:** [templates/source-note.md](../../templates/source-note.md)
- **Resolves / supersedes:** none
