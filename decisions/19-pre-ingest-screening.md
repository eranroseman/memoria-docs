---
mode: explanation
audience: system-designer
topic: decisions
id: 19
title: Pre-ingest screening layer (PRISMA + ASReview)
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-19: Pre-ingest screening layer (PRISMA + ASReview)

## Context

Pre-ingest screening differs from `find` in scale (200–5000 candidates vs. 10–200), trigger (one-time vs. ongoing), and eligibility formality (explicit PRISMA criteria vs. implicit relevance). For formal scoping or systematic reviews, a structurally separate workflow is needed.

## Decision

**Adopt when starting a formal scoping or systematic review.** Use ASReview for ranking; export existing vault papers as priors via `hermes run export prior-labels`. Keep the two pipelines (find vs. pre-ingest screening) separate but share the candidate frontmatter format from [ADR-21](21-shared-candidate-frontmatter.md).

## Consequences

- Casual research doesn't carry PRISMA-screening overhead.
- Systematic reviews have the right tool when they need it.
- Shared candidate frontmatter (ADR-21) means a single Dataview query covers both pipelines.

## Alternatives considered

**Use `find` for both**: rejected — `find` is optimized for low-volume ongoing leads, not 500-candidate one-time database exports. The workflow shapes are different.

**Build a unified pipeline**: rejected — would dilute both. PRISMA needs explicit criteria; find thrives on implicit relevance.

## Related

- **Related decisions:** [ADR-12 systematic-review mode](12-systematic-review-mode.md), [ADR-21 shared candidate frontmatter](21-shared-candidate-frontmatter.md), [ADR-20 dual-rater workflow](20-dual-rater-workflow.md).
- **Workflows affected:** [Find](../workflows/README.md)
- **Files affected:** [profiles/librarian.md](../profiles/librarian.md)
- **Resolves / supersedes:** none

<!-- memoria-nav -->

---

[← Previous: ADR-18: Evidence quality fields layer](18-evidence-quality-fields.md)

[Next: ADR-20: Dual-rater workflow for inter-rater reliability →](20-dual-rater-workflow.md)
