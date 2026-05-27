# Writer — design summary

**Runtime contract.** Full prompt and operational details live at `.memoria/profiles/memoria-writer/SOUL.md` in the starter vault.

## Mission

Writer turns evidence into structured prose — synthesis drafts, wiki-ready content, manuscript sections, and counter-outlines. The defining trait is **drafts, not canonical content**: every Writer output lands in a review state (`10-inbox/02-synthesis/`, `40-workbench/02-drafts/`, or a `30-synthesis/02-wiki/` draft awaiting approval) and never in `30-synthesis/01-permanent/`. The operator owns canonical synthesis; Writer is the composer whose work the operator reviews, edits, and either promotes or discards.

## What this profile is not

- **Not Socratic.** Socratic asks questions to help the operator think before writing. Writer composes prose after thinking is done. They are sequential — Socratic in Process, Writer in Draft — not interchangeable.
- **Not Verifier.** Writer drafts; Verifier checks. Writer's job is to make tracing *possible* (cite explicitly, link to claim notes by wikilink); the actual citation check, claim trace, and similarity check are Verifier's commands. Writer must not pre-empt them.
- **Not Cartographer.** Cartographer maps the corpus; Writer composes arguments from it. Writer reads Cartographer's outputs (`corpus-map.md`, `gap-report.md`) as context, but does not produce maps.
- **Not autonomous about canonical promotion.** Writer can *propose* promotion of a claim-note to reference-note via the `promote` handoff command, but cannot move it. The operator approves the move.

## Design decisions

- **Canonical writes degrade to `dry_run`.** Writer's lane-override declares writes to `30-synthesis/01-permanent/` and `50-deliverables/` as `dry_run` — the writes don't fail loudly, they become board comments for the operator to act on. This is the policy-level enforcement of "canonical synthesis is human-owned"; even an aggressive Writer cannot corrupt the canonical layer.
- **Synthesis is generative, end-to-end.** Writer's method class is **generative**: composing prose, structuring arguments, suggesting alternative outlines — none of these have deterministic derivations from inputs. LLM-required throughout, with one exception: the `query` step is deterministic vault search before drafting begins.
- **The `counter-outline` skill is restrictive by design.** When loaded during the Frame stage, `counter-outline` adds policy.deny rules that narrow Writer's write scope to `40-workbench/01-projects/<project>/framing/` only. This is the canonical example of skill-conditional policy: a skill *tightens* the host lane, never loosens it.
- **No external API access.** Unlike Researcher (network-heavy) or Verifier (Zotero/CrossRef for retraction checks), Writer doesn't reach the outside world. Its inputs are entirely the operator's existing vault — sources, claim notes, MOCs. This keeps the cost surface predictable and prevents prompt-injection-via-fetched-content.

## Permissions and commands

Folder permission matrix lives in [02-profiles.md](../02-profiles.md#folder-permission-matrix); the runtime contract lives in the SOUL.md.

## Related

- Workflows: [08-writing-drafting](../04-workflows/08-writing-drafting.md), [16-frame](../04-workflows/16-frame.md), [05-synthesize](../04-workflows/05-synthesize.md), [18-revise](../04-workflows/18-revise.md)
- ADRs: [15 dedicated review-note type](../07-roadmap/decisions/15-dedicated-review-note-type.md), [03 synthesis draft retention](../07-roadmap/decisions/03-synthesis-draft-retention.md)
- Architecture: [01-architecture/why-no-autonomous-synthesis.md](../01-architecture/why-no-autonomous-synthesis.md) — the principle that motivates the `dry_run` degradation on canonical zones.
- Method class: [rationale/computational-methods.md](../rationale/computational-methods.md) — Writer is on the generative side throughout.
