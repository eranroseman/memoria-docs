---
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
- The deferral has no hard trigger — adoption depends on the operator noticing a recurring "I wish I could query this" pain.

## Alternatives considered

**Adopt now**: rejected because every link becomes a maintenance burden. At small corpus sizes, the overhead would dwarf the analytical payoff.

**Adopt at a numeric threshold** (e.g., 500 claim notes): considered but rejected as too rigid. The trigger should be felt need, not a count.

## Related

- **Workflows affected:** [#5 Synthesize](../../04-workflows.md), [#6 Promote to wiki](../../04-workflows.md)
- **Files affected:** [05-notes-folders.md](../../05-notes-folders.md), `00-meta/01-templates/claim-note.md` (in the starter vault)
- **Resolves / supersedes:** ADR-16 ([Contradictions dashboard](16-contradictions-dashboard.md)) depends on this being adopted.
