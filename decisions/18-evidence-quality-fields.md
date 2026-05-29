---
mode: explanation
audience: system-designer
topic: decisions
id: 18
title: Evidence quality fields layer
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-18: Evidence quality fields layer

## Context

Adding `funding`, `coi` (conflict of interest), `risk_of_bias`, `population`, `intervention_type` fields to paper-notes supports systematic-review work. Empirical studies need these for transparency; theoretical and technical work doesn't.

## Decision

**Adopt only when a project protocol or target journal requires it.** Apply only to empirical papers (RCT, controlled-experiment, quasi-experimental, observational, mixed-methods); set `risk_of_bias: NA` for theoretical and technical work. Per-project activation, not baseline schema.

## Consequences

- Casual reading doesn't carry the evidence-quality overhead.
- Systematic reviews have the fields they need when they need them.
- The `NA` convention for non-empirical work means the fields can coexist with theoretical sources without polluting queries.

## Alternatives considered

**Always-on schema**: rejected — most paper-notes wouldn't carry meaningful values.

**Per-project schema branch** (a fork of the paper-note template): rejected — additive fields are cheaper than maintaining two templates.

## Related

- **Related decisions:** [ADR-12 systematic-review mode](12-systematic-review-mode.md), [ADR-19 pre-ingest screening](19-pre-ingest-screening.md), [ADR-20 dual-rater workflow](20-dual-rater-workflow.md) — same domain.
- **Files affected:** `00-meta/03-templates/paper-note.md` (in the starter vault), [vault/README.md](../vault/README.md)
- **Resolves / supersedes:** none

<!-- memoria-nav -->

---

[← Previous: ADR-17: Retriever / Scout as a separate profile](17-retriever-scout-profile.md)

[Next: ADR-19: Pre-ingest screening layer (PRISMA + ASReview) →](19-pre-ingest-screening.md)
