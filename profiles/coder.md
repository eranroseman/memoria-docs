# Coder — design summary

**Runtime contract.** Full prompt and operational details live at `.memoria/profiles/memoria-coder/SOUL.md` in the starter vault.

## Mission

Coder is the documentary front for an external coding agent (Aider, Kilocode, Claude Code, Codex). Its Hermes-side responsibilities are narrow by design: scaffold a `code-note`, commit per task, document what the external agent did and why. The substantive coding work — generating code, debugging logic, restructuring modules — happens in the external agent, not in Memoria. The defining trait is **the two-agent boundary**: Memoria treats the external coding agent as an opaque peer with shared filesystem access, and the human reviews `code-note` updates as the review gate.

## What this profile is not

- **Not the agent that writes code.** Aider, Claude Code, Codex, and Kilocode do that. Coder scaffolds the handoff (the `code-note`), records provenance, and runs git operations. The actual edits land via the external agent's session, not via Coder's prompt.
- **Not orchestration infrastructure.** Coder does not spawn the external agent as a subprocess, does not parse its output, does not feed it instructions through a programmatic API. The two agents coordinate through a markdown handoff (the `code-note`) and a shared filesystem. This is the explicit choice — see [rationale/coder-external-agent.md](../rationale/coder-external-agent.md) for why.
- **Not Linter.** Linter validates structure (schema, link health, file shape) deterministically. Coder produces code-note artifacts and git commits — its method class is *delegated* (the external agent does the LLM work outside Memoria's runtime).
- **Not a synthesizer of project documentation.** Code-notes describe what was built, motivated by which source notes. They are *handoff artifacts*, not write-ups. Writing about the code, the methodology, or the results is Writer's domain — Coder produces the artifact, Writer produces the prose about the artifact.

## Design decisions

- **Two-agent boundary by design, not by accident.** Memoria's Coder doesn't try to be a coding agent because there are already good coding agents and reimplementing them would just create a worse copy. Instead, Coder owns the *connective tissue* between Memoria's audit and review discipline and the external agent's coding capabilities. The vault is the external agent's read-only context; `40-workbench/03-code/` is its write zone; the `code-note` is the handoff.
- **Confined to `40-workbench/03-code/` and project pages.** Coder's writes never touch `20-sources/`, `30-synthesis/01-permanent/`, or any canonical zone — even as a side effect. This is enforced at the policy MCP layer, not just by convention. Code changes that need to update synthesis or sources go through Writer + operator review, not through Coder.
- **Per-task commits, no mega-commits.** Coder's `commit` command commits one logical change per call. This makes the audit trail granular (one card = one commit = one diff to review) and keeps revert scope small.
- **Repo-vs-vault routing rule.** Small scripts that belong to the vault live in `40-workbench/03-code/`; larger projects earn their own repo and live outside the vault, with a `code-note` in the vault as the index. The threshold and the `repo:` frontmatter convention live in [rationale/coder-external-agent.md](../rationale/coder-external-agent.md).

## Permissions and commands

Folder permission matrix lives in [02-profiles.md](../02-profiles.md#folder-permission-matrix); the runtime contract lives in the SOUL.md. Notably narrow: five commands (`code`, `commit`, `revert`, `workspace`, `scaffold`), mostly thin wrappers around git and the code-note template.

## Related

- Workflows: [09-code-artifacts](../04-workflows/09-code-artifacts.md)
- ADRs: [06 code-agent attachment](../07-roadmap/decisions/06-code-agent-attachment.md), [10 code-artifact autopilot](../07-roadmap/decisions/10-code-artifact-autopilot.md)
- Rationale: [coder-external-agent.md](../rationale/coder-external-agent.md) — the canonical doc on the two-agent boundary, the `files.readonlyInclude` convention, and where projects live (vault vs own repo).
- Method class: [rationale/computational-methods.md](../rationale/computational-methods.md) — Coder is *delegated* to an external LLM-driven agent that Memoria treats as an opaque tool.
