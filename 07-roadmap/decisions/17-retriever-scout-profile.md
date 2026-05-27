---
id: 17
title: Retriever / Scout as a separate profile
status: proposed
date_proposed: 2026-05-15
date_resolved:
supersedes: []
superseded_by: []
---

# ADR-17: Retriever / Scout as a separate profile

## Context

Split the Researcher profile into Retriever (broad discovery, candidate generation) and Researcher (ingest, enrichment, classification)? The split would let Retriever scale across more candidates without taxing Researcher's classification model with discovery-shape work.

## Decision

**Defer.** Keep Researcher unified until discovery volume genuinely overwhelms it.

## Consequences

- Researcher handles both `discover` and `ingest` workflows — simpler to configure, single lane policy, single command catalog.
- At high discovery volume (e.g., the [overnight loop](../future-directions.md) running at scale), Researcher's classifier and discovery scoring might compete for compute; if so, split then.
- A future split has a clean shape — Retriever owns `10-inbox/03-candidates/`, Researcher owns `20-sources/` — so the deferral isn't blocking anything.

## Alternatives considered

**Split now**: rejected — one more profile to configure, with no current bottleneck to motivate it.

**Always-unified**: rejected as a long-term answer — at enough scale, the two concerns diverge.

## Related

- **Workflows affected:** [#3 Discover](../../04-workflows.md), [#2 Ingest](../../04-workflows.md)
- **Files affected:** [02-profiles.md](../../02-profiles.md), [profiles/researcher.md](../../profiles/researcher.md)
- **Resolves / supersedes:** none
