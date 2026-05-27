# Workflow 18: Revise

**Group.** Downstream (stage workflow, added 2026-05)
**Goal.** Close the gap-loop: address verification findings (from [#17 Verify](17-verify.md)) before export.

## Pipeline position

Between [#17 Verify](17-verify.md) and export (inside [#8 Writing and drafting](08-writing-drafting.md)).

## Steps

1. The `revise` card opens with the verification report attached.
2. Operator reads the per-claim findings. For each failed trace, three options:
   - **Soften the claim** — rewrite the sentence so it doesn't make a load-bearing assertion. Recommended when the gap is real but the chapter's argument doesn't depend on this claim.
   - **Pursue the gap** — mark the corresponding `gap:` card with reading priority, push the chapter back to the next revision cycle, and read the missing source. Recommended when the claim is load-bearing.
   - **Accept the soft claim** — leave the sentence; tag the chapter with a known-soft-claim list in the project's `notes.md`. Recommended when the gap exists in the literature itself (you've established the claim is contested, not unsupported).
3. After revisions, the operator re-commits, which fires [#17 Verify](17-verify.md) again. Loop continues until verify returns clean (or the operator explicitly marks remaining gaps as accepted-soft).
4. Once verify is clean (or all remaining gaps are accepted-soft), the `revise` card closes and the project card moves to `export`.

## Owners

Human only. This is not a workflow Hermes can execute — gap-loop decisions are operator judgment about which gaps matter for which argument.

## Card lifecycle

Inherited from [#17 Verify](17-verify.md) — the `verify` card lands in one of `verify-needs-revision` or `verify-needs-attention` and becomes operator-owned. Each operator revision commit re-fires #17, which produces a new verification report; the same card iterates until verify returns clean (or operator marks remaining gaps as accepted-soft and manually advances the card to `approved`). Export (step 8 of [#8 Writing and drafting](08-writing-drafting.md)) cannot run while a verify card sits in a non-clean state.

## Command

No CLI command; revisions happen in the editor. The git post-commit hook re-fires [#17 Verify](17-verify.md).

## Why revise is a stage instead of "just keep editing"

Drafts get revised continuously and informally already. The stage isn't about *whether* revision happens — it's about whether the verify→revise loop has *closed* before export. Without the stage, "I exported a draft with three unaddressed verification findings" is a real failure mode the operator commits silently. With the stage, export is blocked on revise's closure.

## Related

- **Previous workflow:** [#17 Verify](17-verify.md)
- **Returns to:** [#8 Writing and drafting](08-writing-drafting.md) (export step)
