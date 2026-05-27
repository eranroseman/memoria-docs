# Verifier — design summary

**Runtime contract.** Full prompt and operational details live at `.memoria/profiles/memoria-verifier/SOUL.md` in the starter vault.

## Mission

Verifier interrupts the publish→regret loop. It traces every substantive claim in a draft back to a claim note, verifies that every `[@citekey]` resolves to a real source, surfaces near-duplicates before they're filed, and catches retracted sources. The defining trait is **flag, don't fix**: Verifier produces `[!verification]` callouts and verification reports — never edits to drafts. The operator decides whether each flagged claim should be softened, pursued (with a gap card for Researcher to fill), or accepted as-is.

## What this profile is not

- **Not a fact-checker.** Verifier doesn't judge whether a claim is *true*. It judges whether the claim *traces* — whether the citekey resolves, whether the prose has a supporting claim note, whether the similarity to existing claims is suspiciously high. Truth is the operator's domain.
- **Not Linter.** Both check things mechanically, but at different abstraction layers. Linter checks structural integrity (schema, link health, file-system shape) — content-agnostic. Verifier checks content provenance (citations, claim traces, duplicates) — content-aware. They compose: Linter says "this note's schema is valid"; Verifier says "this draft's claims trace to real sources."
- **Not Writer.** Verifier never edits drafts. When a claim fails to trace, Verifier spawns a gap card in the upstream queue (Researcher's problem) and records the failure in the verification report. The draft itself is untouched.
- **Not an LLM-as-judge.** With one carefully-bounded exception (the ambiguous middle band of citation-to-claim matching), Verifier's checks are deterministic — regex extraction, embedding similarity, DOI lookups, set arithmetic. The design avoids "ask the LLM if these match" because that would make the verification step itself non-reproducible.

## Design decisions

- **Read-only across the vault.** Verifier's only write paths are `40-workbench/01-projects/*/verification/*` (the verification reports themselves) and `10-inbox/03-candidates/gap-<slug>.md` (gap cards that Researcher picks up). Drafts, claim notes, and source notes are read-only — Verifier cannot edit the thing it's verifying.
- **Mechanical first, interpretive never.** All five sub-checks (citation, claim-trace, duplicate, retraction, source-note completeness) are deterministic by construction. The one hybrid step — ambiguous-band claim-to-source matching — has explicit threshold tuning: auto-clean above ~0.75 similarity, auto-fail below ~0.4, LLM-judge only the middle band. This bounds where nondeterminism enters the pipeline.
- **Filing-time similarity is informational, never blocking.** A `similarity-check` finding flags the card with `near-duplicate-candidate` and surfaces the top 3 similar claims, but does not block filing. Operator decides between file / merge / extend. Auto-merge is never an option; collapsing two claim notes is a synthesis decision, not a structural one.
- **Gap cards close the loop.** Every failed claim-trace produces a card in `10-inbox/03-candidates/` with `type: gap-candidate` and a backlink to the verification report. Researcher picks these up at the next discovery pass. The verification report is not just a record of failures — it's the spec for Researcher's next round of work.

## Permissions and commands

Folder permission matrix lives in [02-profiles.md](../02-profiles.md#folder-permission-matrix); the runtime contract (the five sub-checks, threshold values, hybrid-band rationale) lives in the SOUL.md.

## Related

- Workflows: [17-verify](../04-workflows/17-verify.md), [08-writing-drafting](../04-workflows/08-writing-drafting.md), [11-merge-split-prune](../04-workflows/11-merge-split-prune.md)
- ADRs: [20 dual-rater workflow](../07-roadmap/decisions/20-dual-rater-workflow.md), [18 evidence quality fields](../07-roadmap/decisions/18-evidence-quality-fields.md)
- Method class: [rationale/computational-methods.md](../rationale/computational-methods.md) — Verifier sits on the hybrid side with a tightly-bounded LLM step; the design explicitly avoids LLM-as-similarity-judge in the determined bands.
- Reference: [reference/computational-toolbox.md](../reference/computational-toolbox.md) — the embedding, similarity, and citation-extraction primitives Verifier relies on.
