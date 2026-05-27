# Architecture Decision Records

This folder holds Memoria's Architecture Decision Records (ADRs) — one file per non-trivial design choice the implementer needs to be aware of. Each ADR follows the template in [`_template.md`](_template.md).

**Status vocabulary** (deliberately small to force clarity):

- `proposed` — under discussion; no action taken yet.
- `accepted` — decided; the codebase follows this rule.
- `superseded` — replaced by a later ADR (see `superseded_by` field).
- `retired` — withdrawn without replacement (e.g., the problem dissolved into another doc, or the deferral lost its trigger).

## Index of all decisions

Resolved decisions that don't have their own file are noted in the **Resolution** column; their historical context is preserved in this index. Decisions with their own ADR file are linked.

| # | Title | Status | Resolution / location |
|---|---|---|---|
| 1 | Reviewer profile vs. human-only review | `accepted` (2026-05) | Index-only. Neither exactly — mechanical checks live in **Verifier** (workflow #17); judgment is always operator-driven. Verifier translates `verify-clean` / `verify-needs-revision` / `verify-needs-attention` into the operator's `approve` / `reject` / `escalate` decisions. |
| 2 | [Auto-promotion threshold](02-auto-promotion-threshold.md) | `proposed` | Manual flagging via `maturity: evergreen`; no auto-promotion to wiki. |
| 3 | [Synthesis-draft retention](03-synthesis-draft-retention.md) | `proposed` | Surface in weekly dashboard; linter flags drafts >90 days for operator decision. |
| 4 | [Citekey naming convention](04-citekey-naming-convention.md) | `accepted` (2026-05) | `authoryearword` via Better BibTeX format string. Pin keys immediately on import. |
| 5 | [MOC depth](05-moc-depth.md) | `retired` (2026-05) | Folded into [linking-patterns MOC thresholds](../../reference/linking-patterns.md#moc-creation-thresholds). |
| 6 | [Code agent attachment](06-code-agent-attachment.md) | `accepted` (2026-05) | Delegate to external coding agent (Claude Code, Aider, Codex, Kilocode); Coder profile scaffolds + documents. |
| 7 | [Session log granularity](07-session-log-granularity.md) | `retired` (2026-05) | Per-session log files in `00-meta/04-logs/`; convention already settled by deployment options. |
| 8 | Hook profile | `accepted` (2026-05) | Index-only. Three tiers — `strict` (default; propose-only), `standard` (safe auto-fixes + low-stakes triage), `minimal` (scheduled synthesis drafting). Named for security posture, not opaque integers. |
| 9 | [Typed `relations:` frontmatter](09-typed-relations-frontmatter.md) | `proposed` | Defer. Plain wikilinks until corpus density justifies the maintenance. |
| 10 | [Code-artifact autopilot](10-code-artifact-autopilot.md) | `proposed` | Defer. Manual triggers until a specific recurring analysis makes the case. |
| 11 | Confidence scoring on `_draft_classification` | `accepted` (2026-05) | Index-only. Multi-label classifier trained on operator's past decisions; LLM fallback below 0.85 confidence. Resolved via [computational-methods classifier-with-LLM-fallback](../../rationale/computational-methods.md). |
| 12 | [Systematic-review mode](12-systematic-review-mode.md) | `proposed` | Adopt only when actively running a systematic review. |
| 13 | [Method-unit vocabulary](13-method-unit-vocabulary.md) | `retired` (2026-05) | Premature ontology; no triggering pattern emerged. |
| 14 | [Cross-run skill-insights memory](14-cross-run-skill-insights.md) | `proposed` | Defer. Significant architecture for a single-user vault. |
| 15 | [Dedicated review-note type](15-dedicated-review-note-type.md) | `proposed` | Defer. Card's review_state / handoff_note carries enough provenance. |
| 16 | [Contradictions / tensions dashboard](16-contradictions-dashboard.md) | `proposed` | Depends on ADR-9 being adopted. |
| 17 | [Retriever / Scout as a separate profile](17-retriever-scout-profile.md) | `proposed` | Keep Researcher unified until discovery volume overwhelms it. |
| 18 | [Evidence quality fields layer](18-evidence-quality-fields.md) | `proposed` | Per-project activation when a protocol or journal requires it. |
| 19 | [Pre-ingest screening layer (PRISMA + ASReview)](19-pre-ingest-screening.md) | `proposed` | Adopt when starting a formal scoping or systematic review. |
| 20 | [Dual-rater workflow for inter-rater reliability](20-dual-rater-workflow.md) | `proposed` | Activate only when the chapter / paper requires it. |
| 21 | [Shared candidate frontmatter format](21-shared-candidate-frontmatter.md) | `accepted` (2026-05) | Adopt the `type: candidate` schema for all candidate sources. Adds `candidate-note` as the 16th canonical note type. |

## Adding a new ADR

1. Copy [`_template.md`](_template.md) to `NN-kebab-case-title.md` where `NN` is the next available integer.
2. Fill in the frontmatter (id, title, status, date_proposed).
3. Write the Context / Decision / Consequences / Alternatives / Related sections.
4. Add a row to the index table above.
5. If this ADR supersedes another, set `superseded_by` on the old one and `supersedes` on the new one.

## Closing an ADR

When a `proposed` decision is acted on:

1. Update `status` to `accepted` (or `retired` if the problem dissolved).
2. Set `date_resolved`.
3. Update the index table's status column.

If a later ADR replaces this one, set the old ADR's `status` to `superseded` and link to the replacement via `superseded_by`. Both files stay — superseded ADRs are valuable history.
