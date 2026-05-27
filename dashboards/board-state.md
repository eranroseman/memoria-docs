# `board-state` — design summary

**Runtime artifact.** This dashboard ships at `00-meta/05-dashboards/board-state.md` in the [starter vault](https://github.com/eranroseman/memoria-vault) and runs in Obsidian via Dataview. The summary below covers its design role; the runtime queries live in the vault file.

## Mission

A Dataview view of cards on the Hermes Kanban board: active cards, review queue, retry watch, claim-note maturity histogram. The board is the system's control plane; this dashboard is the operator's window into it from inside Obsidian. Open when the Hermes Workspace board view isn't convenient — most often, when the operator is already in Obsidian working with notes and wants to see card state without context-switching.

## What this dashboard is not

- **Not the canonical board.** The canonical board lives in Hermes (or in `00-meta/board/` as markdown cards, depending on configuration). This dashboard is a *read view*; state changes happen through Hermes commands or by editing the card files directly, not from inside this dashboard.
- **Not [`index`](index.md)'s "today's queue" section.** Index filters to `blocked-on-human` and `awaiting-review` only, capped at 10. `board-state` shows the full board across all states.
- **Not [`process-queue`](process-queue.md).** Process-queue is upstream-cognitive-discipline (literature notes triaged but not Socratically processed); board-state is workflow-execution (cards moving through states regardless of content type).

## Design decisions

- **Markdown-backed only.** The queries assume the board is markdown-backed (cards as notes in `00-meta/board/`) or exported to markdown. If Hermes is the sole source of truth and there's no markdown export, this dashboard is empty by design — Hermes Workspace is the right surface in that case.
- **Three sections, three failure modes.** Active cards (work-in-flight visibility), review queue (who owes what review), retry watch (which cards are accumulating retries). Each one names a distinct operator concern.
- **The claim-note maturity histogram is an end-of-board signal.** It tracks the downstream output of all the upstream board work — how many claim notes have advanced from `seedling` to `budding` to `evergreen`. A board that flows work without producing maturity is a board not advancing the knowledge graph.

## Related

- [03-board.md](../03-board.md) — Kanban state machine, lane definitions, review gate
- [`process-queue`](process-queue.md) — upstream-discipline view (literature notes awaiting Socratic processing)
- [`index`](index.md) — daily health glance, includes today's queue (filtered subset of board-state)
- [`audit-log`](audit-log.md) — per-decision forensics complementing board-level state
