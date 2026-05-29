---
mode: explanation
audience: operator
topic: board
---

# Board, states, and the review gate

The Kanban board is the control plane for Memoria. It is the shared state machine across profiles and sessions. Every long-lived piece of work lives on the board until a human approves it into the vault.

This document covers the conceptual model: why the board exists, what the states mean, and how cards move through review and retry. For the operational reference — the full state list, lane exit contracts, review-gate rules, and verdict vocabulary — see [states.md](states.md). For the card schema and handoff format see [card-schema.md](card-schema.md).

## Why every state matters

- **`unspecified` vs `queued`** — separates specification from dispatch. A card in `unspecified` may be missing `lane`, `canonical_target`, or a coherent task description. Promotion to `queued` is an explicit "I'm done thinking about scope" moment that the dispatcher waits for. The closest Hermes-default equivalent is the Triage state, but driven by human action rather than an orchestrator — see [Cards born in `unspecified` vs cards born in `queued`](#cards-born-in-unspecified-vs-cards-born-in-queued) below.
- **`queued` vs `claimed`** — separates dispatch from execution. Multiple cards can be `queued`; only one profile holds `claimed` at a time per card.
- **`blocked-on-human` vs `delivered`** — both block, but for different reasons. `blocked-on-human` is a decision the worker cannot make at all (the work itself can't proceed); `delivered` is a check on already-completed work.
- **`declined` vs `requeued`** — `declined` is a quality decision (the work was wrong); `requeued` is an execution problem (a tool failed, a query timed out). `declined` is human-driven and exits via discard-or-supersede; `requeued` is dispatcher-driven and re-attempts the same card.
- **`accepted` vs `closed`** — separation matters when promotion involves moving files or running follow-on tasks. `accepted` says "the work is good"; `closed` says "everything downstream has been handled."

The key discipline: cards never close on a worker's say-so. The card lives until the human changes the review state.

## Cards born in `unspecified` vs cards born in `queued`

The default for **human-created cards** is `unspecified`. The command-palette commands that create cards (`Memoria: new project`, `Memoria: ingest source` when invoked manually, etc.) write the card with `status: unspecified` so the human can review the auto-populated fields and adjust before dispatch.

The default for **cron-created cards and watcher-triggered cards** is `queued`. These cards are pre-specified by their cron config or trigger rule — there is no half-formed state to recover from, and the design intent is fire-and-forget. The nightly hygiene sweep, the weekly drift detector, file-system watchers on `library.bib`, and git-hook-fired verify cards all skip `unspecified` entirely.

The rule: **if the card is fully specified at creation time, start at `queued`; if a human still needs to shape it, start at `unspecified`.** The dispatcher ignores `unspecified` cards regardless of how they got there, so the distinction is visible only in the human's UI.

## Post-rejection paths

A card in `declined` is not "back to the worker." The human's judgment is "no, this work isn't right." What happens next is a separate decision made by the human with full context:

| Path | What happens | Outcome marker |
| --- | --- | --- |
| **Supersede** | Human spawns a new card on the same lane with revised specification — addressing the issues that caused rejection. The new card carries a `supersedes: <original-card-id>` provenance field; the original card closes. This is the standard "revise and retry" pattern. | Original card → `closed` with `outcome: superseded`. New card starts in `unspecified`. |
| **Discard** | Human decides the work shouldn't exist. The card closes without a successor. The rejection itself is the answer. | Card → `closed` with `outcome: discarded`. |

Both paths are explicit human actions. There is no implicit "return to lane queue" — every rework starts as a new card with new specs, which is more honest about how revisions actually work (the original prompt was usually wrong, not just the work product).

This is *not* the same as `requeued`. Requeued is automatic re-dispatch after a transient failure on the same card with the same task packet. Rejection is human judgment that the task itself needs to be re-specified, which is what a new card is for.

## Persistence pattern

The board is where work memory lives across sessions and across worker handoffs.

- **Same card, same identity.** A retry does not create a new card.
- **Session-safe memory.** Closing your chat does not lose the work; the card persists.
- **No chat-history dependence.** The next worker reads the card, not the transcript.
- **Visible until closed.** A card in any non-terminal state is on the board.
- **Shared memory across roles.** The handoff note is the API between profiles.

## Retry pattern

Failed work moves to `requeued` instead of disappearing. The next attempt happens on the same card.

The retry comment should capture:

- **Failure reason.** What broke (tool error, API timeout, unclear input).
- **What was attempted.** Which approaches the previous worker tried.
- **Next action.** The exact thing the next claim should try.

`retry_count` increments. The same or a different profile can reclaim.

### Escalation threshold

**Default: `retry_count > 3` auto-moves the card to `blocked-on-human` with `blocked_reason: "retry_threshold_exceeded"`.** Three attempts is the operating point — enough that transient failures (a rate-limited API, a flaky external lookup) resolve themselves on retry, few enough that a structurally broken task doesn't burn the lane on the same failure mode all night. The threshold is configurable per lane in the lane-override file for lanes where the cost profile justifies a different number — the library lane may tolerate `>5` because each retry is cheap, the Writer lane may want `>2` because each retry is a model call. The Kanban dispatcher reads the threshold from the lane-override; no profile is "responsible" for enforcing it.

The card stays in `blocked-on-human` until the human either revises the task packet (and resets `retry_count` to 0), promotes the issue out of band (rewriting the prompt, fixing a tool), or closes the card with `blocked_reason: "infeasible"`. The retry counter never auto-decrements; only an human decision resets it.

This avoids three problems: duplicate cards for the same work, lost history about why a task is hard, and the temptation to give up silently. The escalation threshold adds a fourth guard: a brittle prompt or a broken tool can't quietly burn through API budget overnight.

## Cross-role flow

A typical card moves through:

```text
Trigger creates card ──► Specialist executes ──► Verifier / Linter recommendation ──► Human approves
                                                                                            │
                                                                                            ├─ approved ──► Done
                                                                                            │
                                                                                            └─ rejected ──► Done (outcome: superseded or discarded)
                                                                                                                  │
                                                                                                                  └─ (optional) new pending card with provenance
```

Triggers (human action, cron, git hook, file watcher) create cards according to the lane-override rules. Specialists act within their lane. Verifier produces a verification verdict on `verify` cards; Linter produces structural reports. The human approves or blocks.

## Board implementation: Hermes built-in Kanban

Memoria uses the **Hermes built-in Kanban board**. This is mandated — not one option among several.

- [Hermes Kanban feature](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Kanban tutorial](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial)
- [Worker lanes](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes)

Why Hermes native:

- The whole architecture assumes Hermes profiles; the native board gives profiles first-class lane semantics without bridging.
- Worker-lane contracts are built in — `delegate_task` for one-off children, profile binding for persistent lanes.
- Review states are native fields, not extensions of a generic board schema.
- Cross-session persistence and retry semantics match what's specified in the schema fields above.

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
- **Not a chat log.** Conversation context goes into handoff notes, not card history.
- **Not a place for canonical claims.** Claims live in `30-synthesis/01-claims/`. The board references them; it doesn't hold them.
- **Not a substitute for review.** A card with `review_status: approved` is meaningful; a card with no review state set is not.

## Anti-patterns

- **Closing cards on worker say-so.** Always wait for the review state to change.
- **Creating a new card for a retry.** Reuse the existing card; increment `retry_count`.
- **Burying review status in comments.** If `review_status` is the source of truth, queries must use it; if comments are the source of truth, review is implicit and unenforceable.
- **Unbounded lanes.** Without WIP limits, the synthesis or review queue fills up and the human becomes the bottleneck silently.

<!-- memoria-nav -->

---

[← Previous: Capability stack](../architecture/capability-stack.md)

[Next: Board states and review gate (reference) →](states.md)
