---
mode: explanation
audience: operator
topic: board
---

# Board, states, and the review gate

The Kanban board is the control plane for Memoria. It is the shared state machine across profiles and sessions. Every long-lived piece of work lives on the board until a human approves it into the vault.

This document covers the conceptual model: why the board exists, what the states mean, and how cards move through review and retry. Memoria tracks **two lifecycles** — *execution* (the Hermes built-in `status` enum) and *review* (a Memoria overlay in the card `metadata`). For the operational reference — the full state machine, lane exit contracts, review-gate rules, and verdict vocabulary — see [states.md](states.md). For the card schema and handoff format see [card-schema.md](card-schema.md). When a word here carries more than one sense — `review`, `verdict`, `promote`, `lane`, `canonical` — the [glossary](../glossary.md) pins which is meant.

## Why every distinction matters

Memoria's earlier drafts invented one nine-value `status` enum. Those distinctions are still meaningful — they just split across the two lifecycles. Each pairing below is worth keeping:

- **`triage` vs `ready`** — separates specification from dispatch. A card in `triage` may be missing an `assignee`, a `promote_target`, or a coherent task description. `hermes kanban specify` turns it into a concrete spec (`triage → todo`); releasing that spec to `ready` is the explicit "I'm done thinking about scope" moment that the dispatcher waits for. This is the Hermes Triage state, driven by human action rather than an orchestrator — see [Cards born in `triage` vs cards born in `ready`](#cards-born-in-triage-vs-cards-born-in-ready) below.
- **`ready` vs `running`** — separates dispatch from execution. Multiple cards can be `ready`; a profile holds one `running` card at a time.
- **`blocked` vs awaiting review** — both stall the card, for different reasons. `blocked` is a decision the worker cannot make at all (the work itself can't proceed); a `done` card with `review_status: requested` is a check on already-completed work.
- **`review_status: rejected` vs auto-retry** — `rejected` is a quality decision (the work was wrong); a retry is an execution problem (a tool failed, a query timed out). Rejection is human-driven and exits via discard-or-supersede; a retry is dispatcher-driven and re-attempts the same card by returning it to `ready`.
- **`review_status: approved` vs `status: archived`** — separation matters when promotion involves moving files or running follow-on tasks. `approved` says "the work is good"; `archived` says "everything downstream has been handled."

The key rule: cards never reach canonical on a worker's say-so. The card lives until the human changes the review state.

## Cards born in `triage` vs cards born in `ready`

The default for **human-created cards** is `triage`. The command-palette commands that create cards (`Memoria: new project`, `Memoria: ingest source` when invoked manually, etc.) write the card with `status: triage` so the human can review the auto-populated fields and adjust before dispatch.

The default for **cron-created cards and watcher-triggered cards** is `ready`. These cards are pre-specified by their cron config or trigger rule — there is no half-formed state to recover from, and the design intent is fire-and-forget. The nightly hygiene sweep, the weekly drift detector, file-system watchers on `library.bib`, and git-hook-fired verify cards all skip `triage` entirely.

The rule: **if the card is fully specified at creation time, start at `ready`; if a human still needs to shape it, start at `triage`.** The dispatcher ignores `triage` cards regardless of how they got there, so the distinction is visible only in the human's UI.

## Post-rejection paths

A card with `review_status: rejected` is not "back to the worker." The human's judgment is "no, this work isn't right." What happens next is a separate decision made by the human with full context:

| Path | What happens | Archive marker |
| --- | --- | --- |
| **Supersede** | Human spawns a new card on the same lane with revised specification — addressing the issues that caused rejection. The new card carries a `metadata.supersedes: <original-card-id>` provenance field; the original card is archived. This is the standard "revise and retry" pattern. | Original card → `archived` with `metadata.archive_reason: superseded`. New card starts in `triage`. |
| **Discard** | Human decides the work shouldn't exist. The card is archived without a successor. The rejection itself is the answer. | Card → `archived` with `metadata.archive_reason: discarded`. |

Both paths are explicit human actions. There is no implicit "return to lane queue" — every rework starts as a new card with new specs, which is more honest about how revisions actually work (the original prompt was usually wrong, not just the work product).

This is *not* the same as a retry. A retry is automatic re-dispatch after a transient failure on the same card with the same `metadata` payload. Rejection is human judgment that the task itself needs to be re-specified, which is what a new card is for.

## Persistence pattern

The board is where work memory lives across sessions and across worker handoffs.

- **Same card, same identity.** A retry does not create a new card.
- **Session-safe memory.** Closing the chat does not lose the work; the card persists in `kanban.db`.
- **No chat-history dependence.** The next worker reads the card, not the transcript.
- **Visible until archived.** A card in any non-terminal state is on the board.
- **Shared memory across roles.** The handoff summary (and the `metadata` payload) is the API between profiles.

## Retry pattern

Failed work returns to `ready` for re-dispatch instead of disappearing. The next attempt happens on the same card.

The retry comment should capture:

- **Failure reason.** What broke (tool error, API timeout, unclear input).
- **What was attempted.** Which approaches the previous worker tried.
- **Next action.** The exact thing the next claim should try.

Hermes increments the retry count in run history. The same or a different profile can reclaim.

### Escalation threshold

**Default: after `max_retries` (default 3) recoverable failures, the dispatcher moves the card to `blocked` with `reason: "retry_threshold_exceeded"`.** Three attempts is the operating point — enough that transient failures (a rate-limited API, a flaky external lookup) resolve themselves on retry, few enough that a structurally broken task doesn't burn the lane on the same failure mode all night. `max_retries` is configurable per lane in the lane-override file for lanes where the cost profile justifies a different number — the library lane may tolerate `5` because each retry is cheap, the Writer lane may want `2` because each retry is a model call. The Kanban dispatcher reads the threshold from the task/lane config; no profile is "responsible" for enforcing it.

The card stays in `blocked` until the human either revises the handoff payload (`metadata`) and re-dispatches (which resets the retry count), promotes the issue out of band (rewriting the prompt, fixing a tool), or archives the card with `reason: "infeasible"`. The retry counter never auto-decrements; only a human decision resets it.

This avoids three problems: duplicate cards for the same work, lost history about why a task is hard, and the temptation to give up silently. The escalation threshold adds a fourth guard: a brittle prompt or a broken tool can't quietly burn through API budget overnight.

## Cross-role flow

A typical card moves through:

```text
Trigger creates card ──► Specialist executes ──► Verifier / Linter recommendation ──► Human reviews
                                                                                          │
                                                                                          ├─ review_status: approved ──► archived
                                                                                          │
                                                                                          └─ review_status: rejected ──► archived (archive_reason: superseded or discarded)
                                                                                                                           │
                                                                                                                           └─ (optional) new triage card with provenance
```

Triggers (human action, cron, git hook, file watcher) create cards according to the lane-override rules. Specialists act within their lane. Verifier produces a verification verdict (in `metadata.agent_verdict`) on `verify` cards; Linter produces structural reports. The human approves or blocks.

## Board implementation: Hermes built-in Kanban

Memoria uses the **Hermes built-in Kanban board**. Once a board is adopted, this is the mandated choice — not one option among several. (The [minimum-viable system](../roadmap/README.md#minimum-viable-system) can run Hermes terminal-only and defer the board until task volume justifies it; what's mandated is *which* board to adopt, not that one is adopted from the start.)

- [Hermes Kanban feature](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Kanban tutorial](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial)
- [Worker lanes](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes)

Why Hermes native:

- The whole architecture assumes Hermes profiles; the native board gives profiles first-class lane semantics (a lane *is* a card's `assignee`) without bridging.
- Worker-lane contracts are built in — `delegate_task` (the `delegation` toolset) for one-off isolated children, profile binding via `assignee` for persistent lanes.
- Cross-session persistence and retry semantics are native — card state lives in `kanban.db` and survives restarts, and `max_retries` drives re-dispatch.
- The one thing Hermes lacks — a human review gate — layers cleanly onto the native `metadata` field, so Memoria adds it without forking the fixed card schema. See [card-schema.md](card-schema.md).

### Dispatch interval

The Hermes Kanban dispatcher polls the queue every **60 seconds** (`dispatch_in_gateway: true`, `dispatch_interval_seconds: 60`). This is the mandated Memoria setting, not a default to be retuned per deployment.

- **Faster (e.g., 10s)** wastes resources — most cards aren't dispatchable on most loops; the polling overhead grows linearly with no corresponding throughput gain.
- **Slower (e.g., 5 min)** makes the system feel laggy — the human triggers a workflow from the command palette and waits visibly long enough to wonder if it landed.

The 60-second cadence is also what makes the async / Kanban surface feel ambient: dropped PDFs get picked up within a minute (fast enough that the human forgets they triggered it); cron-created cards reach their lanes promptly without consuming idle cycles. Don't change it without a specific reason; if a specific lane needs faster response, push the priority of its cards rather than tightening the global interval.

### Alternatives considered, not adopted

| Alternative | Why not |
| --- | --- |
| Obsidian Kanban plugin (via [hermes-kanban](https://github.com/GumbyEnder/hermes-kanban) bridge) | Vault-native and markdown-portable, but requires bridging Hermes worker semantics to the plugin's data model. The bridge works but adds a translation layer between two state machines. |
| GitHub Projects | Strong for code workflows but lives outside the vault and outside Hermes; would require the board state to be in a third place. |
| Linear | Cross-team strength is irrelevant for a single-user system. |
| Plain markdown + Dataview | Minimal, but reimplements Kanban semantics the native board already provides. |

For implementation patterns and a working example of Hermes-managed boards, see also the [KHAOSS-STACK](https://github.com/heimdallthegatekeeper1/KHAOSS-STACK) repo.

## What the board does not do

- **Not a knowledge store.** Cards die; knowledge lives in the vault.
- **Not a chat log.** Conversation context goes into handoff summaries, not card history.
- **Not a place for canonical claims.** Claims live in `30-synthesis/01-claims/`. The board references them; it doesn't hold them.
- **Not a substitute for review.** A card with `review_status: approved` is meaningful; a card with no review state set is not.

## Anti-patterns

- **Reaching canonical on worker say-so.** Always wait for the review state to change.
- **Creating a new card for a retry.** Reuse the existing card; Hermes re-dispatches it and tracks the retry in run history.
- **Burying review status in comments.** If `review_status` is the source of truth, queries must use it; if comments are the source of truth, review is implicit and unenforceable.
- **Unbounded lanes.** Without WIP limits, the synthesis or review queue fills up and the human becomes the bottleneck silently.

<!-- memoria-nav -->

---

[← Previous: Capability stack](../architecture/capability-stack.md)

[Next: Board states and review gate (reference) →](states.md)
