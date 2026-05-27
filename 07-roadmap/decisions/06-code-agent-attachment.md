---
id: 6
title: Code agent attachment
status: accepted
date_proposed: 2026-05-15
date_resolved: 2026-05-15
supersedes: []
superseded_by: []
---

# ADR-6: Code agent attachment

## Context

Does the Coder profile delegate substantive coding work to an external coding agent (Kilocode, Aider, Claude Code, Codex), or implement coding capabilities itself?

## Decision

**Delegate.** The Coder profile scaffolds `code-note` handoffs with vault context (motivating sources, project links, purpose) and coordinates the review gate. The actual code editing happens in a specialized external agent running as a peer with a shared filesystem. The full setup pattern lives in [rationale/coder-external-agent.md](../../rationale/coder-external-agent.md).

## Consequences

- The Coder profile stays narrow (scaffold + document); doesn't accumulate coding complexity it wasn't designed for.
- Operator can use whichever coding agent fits the project (Claude Code for unfamiliar codebases, Aider for fast diffs, etc.).
- Adds a tool dependency — the operator must install and configure one of the external agents.
- The same parallel-agents-with-shared-filesystem pattern generalizes to rendering agents ([open-design](../../rationale/coder-external-agent.md#the-pattern-generalizes-external-rendering-agents)).

## Alternatives considered

**Coder runs code itself** (in-profile execution): rejected because it conflates research-side Hermes (curating provenance) with code-side tools (editing files, running tests). Mixing the two would either bloat the Hermes side with redundant coding capabilities or leave the code-side underdeveloped relative to specialized tools.

## Related

- **Workflows affected:** [#9 Code artifacts](../../04-workflows.md)
- **Files affected:** [profiles/coder.md](../../profiles/coder.md), [rationale/coder-external-agent.md](../../rationale/coder-external-agent.md), `00-meta/01-templates/code-note.md` (in the starter vault)
- **Resolves / supersedes:** none
