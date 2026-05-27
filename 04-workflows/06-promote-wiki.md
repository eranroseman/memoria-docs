# Workflow 6: Promote to wiki

**Group.** Upstream
**Goal.** Convert stable permanent knowledge into reference-grade reference notes.

## Steps

1. Human marks a claim note as `maturity: evergreen`.
2. Weekly dashboard surfaces the promotion queue.
3. Human moves the file to `30-synthesis/02-wiki/`.
4. Sets `promoted_date`.
5. If applicable, the note becomes a MOC-linked reference page.

## Owners

Human owns the promotion decision and the file move. Hermes flags candidates and can compile draft reference notes for review.

## Example

`receptivity-decreases-under-high-cognitive-load.md` accumulates 4 source citations and 3 cross-links over six months → human marks `maturity: evergreen` → the weekly-dashboard "Promotion queue" surfaces it → human writes a tight intro paragraph framing the claim for reference reading → moves the file to `30-synthesis/02-wiki/receptivity-and-cognitive-load.md` → sets `promoted_date: 2026-11-12` → adds an entry to `[[jitai-design-moc]]` so the new reference page is discoverable from the topic hub.

## Related

- **Previous workflow:** [#5 Synthesize](05-synthesize.md) (claim must reach `maturity: evergreen` first)
- **Auto-promotion policy:** [ADR-2 auto-promotion threshold](../07-roadmap/decisions/02-auto-promotion-threshold.md) — manual only, surfaced via dashboard.
- **MOC creation thresholds:** [reference/linking-patterns.md](../reference/linking-patterns.md#moc-creation-thresholds)
