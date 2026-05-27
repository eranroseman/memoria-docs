# Workflow 17: Verify

**Group.** Downstream (stage workflow, added 2026-05)
**Goal.** Trace every substantive claim in a draft back to a claim note in `30-synthesis/01-permanent/`. Flag unsupported claims for the revise loop.

## Pipeline position

Between draft (inside [#8 Writing and drafting](08-writing-drafting.md)) and [#18 Revise](18-revise.md). Fires automatically on draft commit.

## Steps

1. The operator commits to `40-workbench/02-drafts/<project>/<chapter>.md`. A git post-commit hook creates a `verify` card targeting Verifier.
2. Verifier runs `cite-check`: parses the draft into discrete claims (one per sentence containing a `[@citekey]` or a substantive factual assertion), traces each to claim notes via citekey lookup and similarity search.
3. The output lands as a `[!verification]` callout at the top of the draft (see [06-surfaces.md](../06-surfaces/inline.md)), summarizing total claims / traced / failed. The detailed per-claim report writes to `40-workbench/01-projects/<project>/verification/<chapter>-<date>.md`.
4. For each failed trace, Verifier spawns a `gap:<claim-text>` card in the upstream queue (`10-inbox/03-candidates/` with `type: gap-candidate`). This is the feedback loop that closes downstream back to upstream — see [#3 Discover](03-discover.md) for what happens to gap cards.
5. The `verify` card moves to one of three exit states (`verify-clean`, `verify-needs-revision`, `verify-needs-attention`). The operator decides per claim whether the gap is substantive (needs to be filled) or the claim should be softened.
6. Card moves to [#18 Revise](18-revise.md) if any claim needs operator action, or `approved` if everything traced cleanly.

## Owners

Verifier executes `cite-check`; operator decides per claim; gap cards become Researcher's problem ([#3 Discover](03-discover.md)).

## Card lifecycle

`ready` (git post-commit hook creates the `verify` card targeting Verifier; hook-created cards skip `draft`) → `active` (Verifier claims; Kanban-dispatched within 60s of commit) → exits to one of three states: `verify-clean` (operator advances to export), `verify-needs-revision` (advances to [#18 Revise](18-revise.md)), or `verify-needs-attention` (operator-only judgment; retraction or substantive duplicate finding). Each failed claim-trace also creates a child `gap:<claim-text>` card in the upstream Discover queue — those are independent cards, not state changes on the parent.

## Command

Auto-fired by git hook; manual trigger via `hermes -p memoria-verifier run cite-check --draft <draft-path>`.

## Why verify is a stage instead of part of export

`cite-check` was always in the design, but as part of the export pipeline it ran (or didn't) right at the end — when the operator was already committed to shipping. Promoting it to its own stage between draft and export means gap discovery feeds back into upstream weeks before the deliverable matters, not the day before submission.

## Related

- **Profile:** [profiles/verifier.md](../profiles/verifier.md)
- **Hybrid pattern** (deterministic claim extraction + LLM judgment on ambiguous middle band): [rationale/computational-methods.md](../rationale/computational-methods.md)
- **Previous step:** draft (inside [#8 Writing and drafting](08-writing-drafting.md))
- **Next workflow:** [#18 Revise](18-revise.md)
