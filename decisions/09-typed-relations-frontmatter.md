---
mode: explanation
audience: system-designer
topic: decisions
id: 9
title: Typed relations frontmatter
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-9: Typed `relations:` frontmatter

## Context

Adding typed relationships to claim-note frontmatter — e.g. `supports: [[x]]`, `contradicts: [[y]]`, `uses-method: [[z]]` — would enable graph queries that plain wikilinks cannot answer (contradictions, evidence chains, methodological lineages).

## Decision

**Defer.** Start with plain wikilinks; introduce typed relations only when the corpus is dense enough that the structure pays for the maintenance.

## Consequences

- Claim notes stay simple to write — one section of free-text links, no schema discipline per link.
- Graph queries that need contradiction/support semantics (e.g., [ADR-16 contradictions dashboard](16-contradictions-dashboard.md)) are blocked until this is adopted.
- The deferral has no hard trigger — adoption depends on the human noticing a recurring "I wish I could query this" pain.

## Alternatives considered

**Adopt now**: rejected because every link becomes a maintenance burden. At small corpus sizes, the overhead would dwarf the analytical payoff.

**Adopt at a numeric threshold** (e.g., 500 claim notes): considered but rejected as too rigid. The trigger should be felt need, not a count.

## Related

- **Workflows affected:** [Distill](../workflows/upstream/distill.md), [Promote](../workflows/upstream/promote.md)
- **Files affected:** [vault/README.md](../vault/README.md), `00-meta/03-templates/claim-note.md` (in the starter vault)
- **Required by:** [ADR-16 (Contradictions dashboard)](16-contradictions-dashboard.md) — blocked until this is adopted.

<!-- memoria-nav -->

---

[← Previous: ADR-7: Session log granularity](07-session-log-granularity.md)

[Next: ADR-10: Code-artifact autopilot →](10-code-artifact-autopilot.md)
