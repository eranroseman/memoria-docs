---
mode: explanation
audience: system-designer
topic: decisions
id: 16
title: Contradictions / tensions dashboard
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-16: Contradictions / tensions dashboard

## Context

Surfacing contradictions in the claim graph would be high-value for argument construction — the human could see "claims I've filed that disagree with each other" as a starting point for synthesis. The dashboard depends on typed relations (`relations: { contradicts }`) being populated, which is currently [ADR-9](09-typed-relations-frontmatter.md)'s deferred decision.

## Decision

**Defer.** Adopt only if [ADR-9 (typed relations)](09-typed-relations-frontmatter.md) is adopted. Without typed relations, this dashboard has no data to query.

## Consequences

- Contradictions stay invisible at scale — the human must remember which claims conflict.
- Adoption is gated on ADR-9, which is itself gated on felt need.
- The dependency chain (ADR-16 needs ADR-9, ADR-9 needs corpus density) gives a clear adoption path when the time comes.

## Alternatives considered

**LLM-judged contradictions** (no typed relations needed; let an LLM read the corpus and flag tensions): rejected because LLM-as-similarity-judge has the same calibration problem named in [computational-methods anti-patterns](../architecture/why-computational-methods.md#anti-patterns) — different runs would surface different tensions, with no stable ground truth.

## Related

- **Depends on:** [ADR-9 typed relations](09-typed-relations-frontmatter.md)
- **Files affected:** [`dashboards/`](../dashboards/) (would add a new dashboard)