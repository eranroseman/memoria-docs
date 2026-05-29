---
mode: explanation
audience: operator
topic: operations
---

# Operations

Operational concerns: what to do when things go wrong, and how to keep the running system healthy. The design documents tell you what Memoria *is*; this folder tells you what to *do* when reality diverges from design.

## What lives here

- [failure-modes.md](failure-modes.md) — Detect / Fix / Verify recipes for the common breakages: stuck cards, broken `library.bib`, missing extracts, denied policy actions, profile crashes, dashboard staleness. Open this when something looks wrong but you're not sure what.
- [session-logging.md](session-logging.md) — how the per-session audit trail is written and preserved (the mechanism behind the audit-log dashboard). A system *mechanism*, not a workflow.

## What lives elsewhere

- **Plugin configuration** — [plugins/](../plugins/) (per-plugin configs, lifecycle, visual style). Operationally adjacent but its own folder because of size (20+ files).
- **Deployment and secret management** — [roadmap/deployment-options.md](../roadmap/deployment-options.md), [roadmap/secret-management.md](../roadmap/secret-management.md). These are about *setup* rather than runtime recovery.
- **Safe-mode procedures** — `00-meta/04-reference/safe-mode.md` in the starter vault. The vault-resident counterpart of `failure-modes.md` for when Hermes, the ACP connection, or the watcher is down. See [vault/README.md vault skeleton](../vault/README.md#vault-skeleton-human-facing-notes).
- **Audit and observability** — [dashboards/audit-log.md](../dashboards/audit-log.md), [dashboards/fleet-health.md](../dashboards/fleet-health.md). Dashboards surface the signal; operations recipes act on it.

## Operating principles

- **Recipes follow Detect / Fix / Verify shape.** Every failure-mode entry names a symptom (what you see), a fix (the smallest thing that resolves it), and a verification (how you confirm the fix held). Skipping verify is the most common cause of "I thought I fixed it last week."
- **Prefer reversible actions.** Schema migrations are dry-run first. Auto-fixes for canonical content are denied by policy, not by convention. Operations recipes never instruct the human to run a destructive command without a dry-run path.
- **The board is the operational dashboard.** Stuck cards, retry exhaustion, and review-queue depth are all visible on [board-state](../dashboards/board-state.md) and [discuss-queue](../dashboards/discuss-queue.md). Most operational diagnosis starts there.

<!-- memoria-nav -->

---

[← Previous: weekly-review — design summary](../dashboards/weekly-review.md)

[Next: Failure modes — Detect / Fix / Verify →](failure-modes.md)
