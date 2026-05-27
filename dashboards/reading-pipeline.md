# `reading-pipeline` — design summary

**Runtime artifact.** This dashboard ships at `00-meta/05-dashboards/reading-pipeline.md` in the [starter vault](https://github.com/eranroseman/memoria-vault) and runs in Obsidian via Dataview. The summary below covers its design role; the runtime queries live in the vault file.

## Mission

Keep papers flowing through the upstream pipeline and surface what's stuck. The operator opens this when the inbox feels full and they need to decide what to process next. Two views: papers in active processing (triaged but `partial`), and claim notes by maturity (downstream of processing). Together they answer "what should I read next?" and "what came out of my recent reading?"

## What this dashboard is not

- **Not [`process-queue`](process-queue.md).** Process-queue is narrowly scoped to fully-triaged literature notes that haven't yet had a Socratic pass — the upstream-cognitive-discipline view. Reading-pipeline is broader: any literature note in `partial` state, regardless of triage completeness. Reading-pipeline asks "what's in flight?"; process-queue asks "what owes me a Socratic conversation?"
- **Not [`weekly-dashboard`](weekly-dashboard.md).** Weekly is a ritual entry point; reading-pipeline is a working surface used between rituals.
- **Not a board view.** It queries note state (literature-note `triage_status` and claim-note `maturity`), not card state. [`board-state`](board-state.md) is the card view.

## Design decisions

- **Two queries, two cadences.** "Papers in active processing" answers a near-term question (what to read this session). "Claim notes by maturity" answers a longer-term question (what's the durable output of all this reading). Surfacing both in one dashboard prevents the operator from optimizing one without watching the other.
- **`partial` triage status is the in-flight signal.** A literature note moves from `partial` to `full` when the operator (or Verifier) finishes triage. Reading-pipeline shows everything currently in `partial` — the pipeline's middle band.
- **Sort by `file.mtime` not by created date.** Recency of touch matters more than recency of ingest: a paper triaged 6 months ago and edited yesterday is more likely the operator's current focus than one ingested yesterday but not yet touched.

## Related

- [`process-queue`](process-queue.md) — the narrower upstream-discipline view
- [`weekly-dashboard`](weekly-dashboard.md) — links to reading-pipeline as the weekly reading-planning step
- [04-workflows/14-process.md](../04-workflows/14-process.md) — the Process stage that drains the queue
- [05-notes-folders.md](../05-notes-folders.md) — definitions of `triage_status` and `maturity` lifecycle states
