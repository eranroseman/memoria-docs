# Workflow 13: Session logging

**Group.** Maintenance
**Goal.** Preserve traceability for agent actions.

## Steps

1. Hermes logs the session start.
2. Records work done during the session.
3. Writes per-session log files in `00-meta/04-logs/`.
4. Git captures the history.

## Owners

Hermes writes logs. Git preserves history. Human commits / pushes when needed.

## Related

- **Granularity:** [ADR-7 session log granularity](../07-roadmap/decisions/07-session-log-granularity.md) — per-session files, not per-action.
- **Multi-machine sync:** per-session files survive sync without conflict; see [07-roadmap/sync-and-coordination.md](../07-roadmap/sync-and-coordination.md).
