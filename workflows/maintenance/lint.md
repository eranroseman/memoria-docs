---
mode: how-to
audience: operator
topic: workflows
---

# Lint

**Group.** Maintenance
**Goal.** Keep the vault structurally healthy and queryable.

## Steps

1. The Linter runs lint checks on a schedule or post-ingest.
2. Reports orphans, stale enrichment, partial classification, broken links, schema issues.
3. Human reviews the report.
4. Decides whether to auto-fix or manually correct.

## Owners

Hermes for detection and reporting. Human for approval of fixes.

## Command

`hermes run lint --dry-run`.

## Weekly ritual

A practical weekly cadence that exercises this workflow end-to-end. Approximate time: 30–60 minutes once the vault is established.

| Order | Step | Goal | Approximate time |
| --- | --- | --- | --- |
| 0 | Read `research-directions.md` | Confirm or update the week's priorities. | 2 min |
| 1 | Open `weekly-review.md` | Surface the queues. | < 1 min |
| 2 | Clear **unreviewed synthesis** | Review or discard inbox drafts. | 10–15 min |
| 3 | Process **discovery candidates** | Include or exclude with a reason. | 5–10 min |
| 4 | Resolve **classification debt** | Promote proposed fields; mark `lifecycle: current`. | 10–15 min |
| 5 | Promote **evergreen claim notes** | Move qualifying notes to `30-synthesis/02-reference/`. | 5–10 min |
| 6 | Review **orphan notes** | Add MOC links, concept links, or archive. | 5 min |
| 7 | Inspect **stale literature and items** | Refresh enrichment or archive. | 5 min |
| 8 | Run lint dry-run | Catch structural issues. | 1 min + review |

The ritual isn't a separate workflow — it's the operating cadence that drives this workflow (and touches [Classify](../upstream/classify.md), [Promote](../upstream/promote.md), [Refactor](refactor.md)). The dashboards in [surfaces/README.md](../../surfaces/README.md) are the surfaces; this is the schedule.

## Operating rhythm

This weekly ritual is the maintenance slice of a larger day-to-day rhythm — morning glance, reading and writing sessions, Telegram-driven mobile capture, and the Friday ritual itself. That full rhythm (what using Memoria *feels like* hour-to-hour) lives with the [default operating model](../README.md#default-operating-model).

## Related

- **Profile:** [profiles/linter.md](../../profiles/linter.md)
- **Dashboards:** [drift-watch.md](../../dashboards/drift-watch.md), [weekly-review.md](../../dashboards/weekly-review.md), [Daily Health](../../dashboards/README.md)
- **Standard cron tasks:** [roadmap/standard-cron-tasks.md](../../roadmap/standard-cron-tasks.md)

<!-- memoria-nav -->

---

[← Previous: Code](../downstream/code.md)

[Next: Refactor →](refactor.md)
