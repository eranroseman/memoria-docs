# `index` — design summary

**Runtime artifact.** This dashboard ships at `00-meta/05-dashboards/index.md` in the [starter vault](https://github.com/eranroseman/memoria-vault) and runs in Obsidian via Dataview. The summary below covers its design role; the runtime queries live in the vault file.

## Mission

The always-on **system-health** view, opened every morning. Four sections, each one a one-decision query: today's blocked-on-human and awaiting-review cards, last 24h HIGH/CRITICAL drift signals, lane-health trust scores, and cron status. Glance for 30 seconds. If nothing is red, close it and move on. If something is red, the surfaced item names what needs attention before any larger operation.

## What this dashboard is not

- **Not a vault audit.** Folder counts, orphan notes, stale literature — those are the [`weekly-dashboard`](weekly-dashboard.md)'s Friday ritual job. Index is system health, not knowledge health.
- **Not a task list.** It surfaces decisions waiting on the operator; the operator chooses which to address. The board, not this dashboard, is where state changes happen.
- **Not a substitute for [`drift-watch`](drift-watch.md), [`fleet-observability`](fleet-observability.md), or [`audit-log`](audit-log.md).** The index summarizes the *red signals* from each; the full views live in the deeper dashboards. See ["dashboard-of-dashboards" pattern](#dashboard-of-dashboards-pattern) below.

## Design decisions

- **Dashboard-of-dashboards pattern.** Three of the four index sections are *filtered subsets* of deeper dashboards: drift signals filter `drift-watch` to last 24h HIGH/CRITICAL; lane health filters `fleet-observability` to lane + trust + tasks + success%; today's queue filters `board-state` to `blocked-on-human` / `awaiting-review`. Same data sources, narrower projections. No drift risk between layers because both read the same underlying JSONL/note files.
- **Cron status is unique to index.** No other dashboard surfaces cron run history; this is the one section without a deeper counterpart.
- **30-second contract.** The four sections are designed so a healthy day reads as four empty tables and closes. Anything non-empty is a signal to act on or click through to the deeper view.
- **Capability gates.** Until the metrics aggregator, board-state JSONL feed, lint-findings JSONL feed, and cron-history JSONL feed exist, the four queries return empty. The placeholders state what would populate them, so an empty result is interpretable as "feature not yet wired" rather than "nothing wrong" — see [06-surfaces/persistent.md capability-gates discipline](../06-surfaces/persistent.md#capability-gates-degrade-dont-fail).

## Related

- [`weekly-dashboard`](weekly-dashboard.md) — Friday-ritual vault-state view
- [`drift-watch`](drift-watch.md) — full M1–M8 detector view + verdict band
- [`fleet-observability`](fleet-observability.md) — trust score contributing inputs + cost trends
- [`audit-log`](audit-log.md) — per-decision forensics when something needs investigation
- [`board-state`](board-state.md) — full Kanban board view
