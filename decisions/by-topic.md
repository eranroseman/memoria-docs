---
mode: reference
audience: contributor
topic: general
---

# ADRs grouped by topic

The flat numbered list in [README.md](README.md) is the authoritative order (ADRs are append-only and keep their original number). This file is a *secondary view* that groups the same ADRs by the topic folder they most affect, so a reader exploring `vault/` or `workflows/` can find the related decisions quickly.

Status legend: **proposed** (under discussion) · **accepted** (adopted into the design) · **retired** (superseded or no longer relevant — kept for context).

## Vault & schema

Note types, frontmatter, naming, and the controlled vocabularies that govern them.

- [04-citekey-naming-convention.md](04-citekey-naming-convention.md) — Citekey naming convention · **accepted**
- [09-typed-relations-frontmatter.md](09-typed-relations-frontmatter.md) — Typed relations frontmatter · proposed
- [15-dedicated-review-note-type.md](15-dedicated-review-note-type.md) — Dedicated review-note type · proposed
- [18-evidence-quality-fields.md](18-evidence-quality-fields.md) — Evidence quality fields layer · proposed
- [21-shared-candidate-frontmatter.md](21-shared-candidate-frontmatter.md) — Shared candidate frontmatter format · proposed

## Workflows

Upstream / downstream / maintenance pipelines and their stage-specific design choices.

- [02-auto-promotion-threshold.md](02-auto-promotion-threshold.md) — Auto-promotion threshold · proposed
- [03-answer-draft-retention.md](03-answer-draft-retention.md) — Answer-draft retention · proposed
- [10-code-artifact-autopilot.md](10-code-artifact-autopilot.md) — Code-artifact autopilot · proposed
- [12-systematic-review-mode.md](12-systematic-review-mode.md) — Systematic-review mode · proposed
- [19-pre-ingest-screening.md](19-pre-ingest-screening.md) — Pre-ingest screening layer (PRISMA + ASReview) · proposed
- [20-dual-rater-workflow.md](20-dual-rater-workflow.md) — Dual-rater workflow for inter-rater reliability · proposed

## Profiles

The seven Hermes workers — missions, attachment, cross-run memory.

- [06-code-agent-attachment.md](06-code-agent-attachment.md) — Code agent attachment · **accepted**
- [14-cross-run-skill-insights.md](14-cross-run-skill-insights.md) — Cross-run skill-insights memory · proposed
- [17-retriever-scout-profile.md](17-retriever-scout-profile.md) — Retriever / Scout as a separate profile · proposed

## Dashboards & surfaces

Persistent dashboards and how the human sees board / vault state.

- [16-contradictions-dashboard.md](16-contradictions-dashboard.md) — Contradictions / tensions dashboard · proposed

## Retired

Kept for historical context; superseded or no longer in force.

- [05-moc-depth.md](05-moc-depth.md) — MOC depth · retired
- [07-session-log-granularity.md](07-session-log-granularity.md) — Session log granularity · retired
- [13-method-unit-vocabulary.md](13-method-unit-vocabulary.md) — Method-unit vocabulary · retired

## Gaps

ADR numbers 01, 08, and 11 are reserved/withdrawn (the numbering is append-only and skips reflect drafts that were never adopted). New ADRs take the next unused number; see [`_template.md`](_template.md).

<!-- memoria-nav -->

---

[← Previous: ADR-NN: \<title>](_template.md)

[Next: Implementation roadmap →](../roadmap/README.md)
