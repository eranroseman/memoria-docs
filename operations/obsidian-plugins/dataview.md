# dataview

Load-bearing settings:

- `enableJs: true` — required for the `dataviewjs` blocks in [audit-log.md](../../dashboards/audit-log.md) and [fleet-observability.md](../../dashboards/fleet-observability.md), which read external files (`audit.jsonl`, `lane-metric` notes).
- `refreshEnabled: true` — without this, queries don't update as notes change.
- `refreshInterval: 2500` (ms) — default is fine. Going lower hurts performance on large vaults.
