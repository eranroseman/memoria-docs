---
mode: explanation
audience: operator
topic: operations
---

# Session logging

**Concern.** Observability / audit.
**Goal.** Preserve a durable, traceable record of agent activity.

Session logging is a **system mechanism, not a workflow.** Every agent session writes a per-session log; Git preserves the history. There is no card, nothing to claim, and no state transition — it runs continuously *underneath* every workflow rather than being one (so nothing here can be a "stuck card"). Contrast the maintenance *workflows* — [Lint](../workflows/maintenance/lint.md) and [Refactor](../workflows/maintenance/refactor.md) — which are card-driven. Session logging is the substrate the [audit-log](../dashboards/audit-log.md) and [fleet-health](../dashboards/fleet-health.md) dashboards read.

## How it works

1. Hermes logs the session start.
2. Records work done during the session.
3. Writes per-session log files in `00-meta/02-logs/`.
4. Git captures the history; the human commits / pushes when needed.

## Owners

Hermes writes logs. Git preserves history. Human commits / pushes when needed.

## Related

- **Granularity:** [ADR-7 session log granularity](../decisions/07-session-log-granularity.md) — per-session files, not per-action.
- **Multi-machine sync:** per-session files survive sync without conflict; see [roadmap/sync-and-coordination.md](../roadmap/sync-and-coordination.md).
- **Dashboards that read these logs:** [audit-log](../dashboards/audit-log.md), [fleet-health](../dashboards/fleet-health.md).

<!-- memoria-nav -->

---

[← Previous: Failure modes — Detect / Fix / Verify](failure-modes.md)

[Next: Obsidian plugins — configuration reference →](../plugins/README.md)
