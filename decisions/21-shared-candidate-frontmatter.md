---
mode: explanation
audience: system-designer
topic: decisions
id: 21
title: Shared candidate frontmatter format
status: proposed
date_proposed: 2026-05-15
supersedes: []
superseded_by: []
---

# ADR-21: Shared candidate frontmatter format

## Context

Candidate notes can arrive from three pipelines: `find` (forward/backward citation search), database-search (PRISMA-style bulk screening — see [ADR-19](19-pre-ingest-screening.md)), or manual (human-typed lead). Without a shared schema, each pipeline produces its own slightly different frontmatter and dashboards must run three separate queries.

## Decision

Adopt the unified frontmatter schema for candidate notes:

```yaml
type: candidate
source: find                # find | database-search | manual
candidate_status: pending   # pending | included | excluded
exclusion_reason: ""
projects: []                # plural list, matches other templates
```

`candidate` is not in the 15 note types in [vault/README.md](../vault/templates.md#note-types); adopting this ADR means adding it as a 16th type with its own template (`templates/candidate-note.md`) and updating the list.

## Consequences

- A single Dataview query in the [weekly-review](../dashboards/weekly-review.md) covers all candidate sources.
- Triage dashboards work uniformly regardless of where a candidate came from.
- Tiny schema cost; high payoff for any later screening work.
- Until the 16th note type is added in templates, the dashboard's "Discovery candidates" query returns no results.

## Alternatives considered

**Per-pipeline schemas**: rejected — duplicates effort and forces three parallel queries in every candidate dashboard.

**Hold off until [ADR-19](19-pre-ingest-screening.md) is adopted**: rejected — the shared format pays off for `find` alone (the current primary candidate source), and adopting it later wouldn't be cheaper.

## Related

- **Related decisions:** [ADR-19 pre-ingest screening](19-pre-ingest-screening.md) consumes this schema for bulk screening.
- **Files affected:** [vault/README.md](../vault/README.md), `templates/candidate-note.md` (to be created), [dashboards/weekly-review.md](../dashboards/weekly-review.md)
- **Resolves / supersedes:** none

<!-- memoria-nav -->

---

[← Previous: ADR-20: Dual-rater workflow for inter-rater reliability](20-dual-rater-workflow.md)

[Next: ADR-NN: \<title> →](_template.md)
