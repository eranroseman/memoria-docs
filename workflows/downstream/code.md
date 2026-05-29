---
mode: how-to
audience: operator
topic: workflows
---

# Code

**Group.** Downstream
**Goal.** Treat code as a first-class research output with provenance.

## Steps

1. Human identifies a research code need.
2. The Coder creates or scaffolds a code-note in `40-workbench/01-projects/<project>/code/`.
3. Coder or human develops the code.
4. Output is linked back to the motivating literature.
5. The artifact page records purpose, architecture, and usage.

## Owners

Human owns intent and review. Coder owns implementation. Hermes scaffolds and bookkeeps.

## Example

A chapter needs a reproducible figure → the Coder scaffolds `40-workbench/01-projects/<project>/code/figure-3-receptivity-curve.md` (a `code-note`) linking the motivating claim note → the external coding agent implements the script in the same folder → the human reviews via the standard review gate → the `code-note` records purpose, dependencies, and how to regenerate the figure.

## Related

- **Profile:** [profiles/coder.md](../../profiles/coder.md)
- **External coding agent pattern:** [profiles/why-coder-external-agent.md](../../profiles/why-coder-external-agent.md)
- **Code agent attachment:** [ADR-6 code agent attachment](../../decisions/06-code-agent-attachment.md) — delegate to external agent (Claude Code, Aider, Codex, Kilocode).
- **Autopilot policy:** [ADR-10 code-artifact autopilot](../../decisions/10-code-artifact-autopilot.md) — deferred.
