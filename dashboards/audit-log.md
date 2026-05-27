# `audit-log` — design summary

**Runtime artifact.** This dashboard ships at `00-meta/05-dashboards/audit-log.md` in the [starter vault](https://github.com/eranroseman/memoria-vault) and runs in Obsidian via Dataview. The summary below covers its design role; the runtime queries live in the vault file.

## Mission

The forensic trail for every vault write the policy MCP touched. Spot decisions that need attention: writes blocked at the tool layer (`deny`), dry-run escalations awaiting operator action (`dry_run`), writes to canonical zones (`allow_with_log`). Open when something feels off — a worker behaving strangely, a board card stuck on an unclear reason, or after a scheduled overnight run completes. The audit log file (`00-meta/04-logs/audit.jsonl`, append-only JSONL) is the canonical source; this dashboard is the queryable view.

## What this dashboard is not

- **Not [`drift-watch`](drift-watch.md).** Audit-log records *per-write decisions* (policy MCP outcome per attempted write). Drift-watch records *per-lint-pass findings* (M1–M8 detector outputs). Different cadence, different abstraction.
- **Not a trend view.** [`fleet-observability`](fleet-observability.md) aggregates audit-log entries into rolled-up metrics (audit deny rate, drift incidents, retry rate). Audit-log itself is the raw event stream — one JSON object per write decision.
- **Not editable.** The audit log is append-only by design. Each entry has `before_hash` and `after_hash` SHA-256s for tamper detection; the [Linter's M2 detector](../profiles/linter.md) catches files modified outside this trail.

## Design decisions

- **Reads directly from `00-meta/04-logs/audit.jsonl`** via Dataview's `dv.io.load`. No intermediate aggregation; the dashboard always reflects the latest log state.
- **Recent denies and dry-runs is the action queue.** Sorted newest-first, capped at 30. Anything sitting here for more than a day without a corresponding board card is an unhandled escalation.
- **Canonical-zone writes are surfaced explicitly.** Writes to `30-synthesis/01-permanent/`, `30-synthesis/02-wiki/`, `30-synthesis/03-moc/`, `50-deliverables/` are operationally significant — even when `allow`'d, they should be reviewable. The dashboard's "Canonical-zone writes" section surfaces them for periodic audit.
- **The Linter owns rotation.** Weekly rotation of `audit.jsonl` to `00-meta/04-logs/archive/audit-YYYY-WW.jsonl` is the Linter's responsibility; the dashboard queries the current week's file. Archive files are visible to the operator but not queried by default.

## Related

- [reference/policy-mcp.md](../reference/policy-mcp.md) — the policy MCP that writes these entries; format and decision protocol
- [`drift-watch`](drift-watch.md) — structural drift findings (different layer, complementary view)
- [`fleet-observability`](fleet-observability.md) — operational rollups that consume this stream
- [Linter design summary](../profiles/linter.md) — owns audit-log rotation
