---
mode: reference
audience: operator
topic: dashboards
---

# `discuss-queue` — design summary

**Runtime artifact.** This dashboard ships at `00-meta/01-dashboards/discuss-queue.md` in the [starter vault](https://github.com/eranroseman/memoria-vault) and runs in Obsidian via Dataview. The summary below covers its design role; the runtime queries live in the vault file.

## Mission

Surface every paper note that has been fully classified but hasn't yet had a Socratic processing pass. This is the **upstream-cognitive-discipline dashboard** — if cards pile up here, the human's processing rhythm is slipping. The corollary signal is at the other end: when the list is short, the human's processing is keeping up with their ingest rate. Discuss-queue exists to surface that asymmetry directly, before it becomes a synthesis backlog three months later.

## What this dashboard is not

- **Not [`reading-pipeline`](reading-pipeline.md).** Reading-pipeline is broader (any `partial` paper note in flight); discuss-queue is narrower (specifically `full`-classified notes awaiting Socratic processing). Reading-pipeline asks "what's in the middle band?"; discuss-queue asks "what owes me a Socratic conversation?"
- **Not a generic to-do list.** The implied next action is specifically *Socratic processing* (workflow Discuss) — invoke the Socratic profile, work through the questions, then write a claim note. Surfacing the queue without the implied next action would just be a list.
- **Not Mapper's territory.** Discuss-queue is per-source upstream work. Mapper's outputs (corpus-map, gap-report, comparative-brief) operate across sources at a different abstraction.

## Design decisions

- **Five-or-fewer rows = healthy. Ten or more = schedule a reading session.** These are human-facing health thresholds called out in the dashboard, not enforced by any system. The point is to make the queue's depth read at a glance.
- **`lifecycle: current` AND no `processed:` tag is the gate.** A paper note is on the queue when classification is complete and Socratic hasn't happened yet. Adding a `processed:` task line removes it from the queue.
- **Open during a reading session, not as a glance.** Different cadence from [`index`](README.md). The human opens discuss-queue when they're sitting down to read — not as a daily health-monitor signal.
- **The Reading & Processing workspace surfaces this dashboard.** Per [surfaces/modal.md](../surfaces/modal.md), discuss-queue is the left pane of the Cmd-2 workspace alongside [`reading-pipeline`](reading-pipeline.md). The workspace exists specifically to protect this discipline.

## Related

- [workflows/upstream/discuss.md](../workflows/upstream/discuss.md) — the workflow this dashboard surfaces
- [Socratic design summary](../profiles/socratic.md) — the profile invoked to drain the queue
- [`reading-pipeline`](reading-pipeline.md) — broader sibling view
- [surfaces/modal.md](../surfaces/modal.md) — the Reading & Processing workspace this dashboard anchors

<!-- memoria-nav -->

---

[← Previous: open-questions — design summary](open-questions.md)

[Next: reading-pipeline — design summary →](reading-pipeline.md)
