# Standard cron tasks

Memoria ships with four standard scheduled tasks. Each is declared in the relevant profile's `cron/` folder (see [reference/profile-compilation.md](../reference/profile-compilation.md)) and dispatched by the Hermes Kanban according to the schedule. All four are **read-mostly or write-to-logs only** — they produce reports the operator reads on a cadence, not edits the operator has to review per-task.

```yaml
cron:
  - schedule: "0 2 * * *"          # nightly 02:00
    skill: hygiene-sweep
    lane: linter
    creates_card: {task: nightly-hygiene, state: ready}

  - schedule: "0 3 * * MON"        # weekly Monday 03:00
    skill: cluster-cartography-scan
    lane: cartography
    creates_card: {task: weekly-cluster-report, state: ready}

  - schedule: "0 4 * * MON"        # weekly Monday 04:00
    skill: drift-detector
    lane: linter
    creates_card: {task: weekly-drift-report, state: ready}

  - schedule: "0 5 * * *"          # nightly 05:00
    skill: stale-fleeting-check
    lane: linter
    creates_card: {task: fleeting-staleness-report, state: ready}
```

Three of the four are on the Linter lane (read-only writes go to `00-meta/04-logs/` and dashboard updates) and one is on the Cartography lane (writes a corpus-wide cluster scan to project-scratch). None can modify canonical content — that constraint is what makes scheduled automation safe to enable.

## `cron_mode` migration

Hermes's default config ships `cron_mode: deny` — scheduled tasks don't fire until the operator explicitly enables cron per lane. This is the safety default; Memoria preserves it.

The recommended enablement order, by safety:

1. **Linter first.** All Linter writes go to `00-meta/04-logs/` (audit and session logs) or dashboard files. No content edits, no canonical writes. Lowest blast radius. Enable after Phase 3 (profile build) is stable.
2. **Cartography next.** Writes go to project-scratch (`40-workbench/01-projects/*/corpus-map.md` etc.). No canonical writes. Enable a few weeks after Linter cron is stable.
3. **Researcher (eventually).** Researcher writes to `10-inbox/` and `20-sources/` — actual content, but in zones the operator triages before promotion. Higher blast radius; enable only after the overnight-loop discipline is established (see [future-directions.md — overnight loop](future-directions.md#the-overnight-loop-proactive-discovery-pattern)).
4. **Never auto-enable Writer, Verifier, or Coder.** These produce review-gated artifacts the operator must look at; scheduled cron-dispatch would silently fill the review queue. Always operator-initiated.
5. **Socratic doesn't apply.** Socratic is `routing.invocation: interactive_only` — the Kanban dispatcher won't queue-dispatch it regardless of `cron_mode`.

The discipline: each cron-enable is a deliberate decision recorded in the operator's deployment notes. "Linter cron enabled 2026-06-12 after 4 weeks of stable dry-run reports" is the kind of provenance that makes the system auditable when something goes wrong.
