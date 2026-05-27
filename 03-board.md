# 03 — Board, states, and the review gate

The Kanban board is the control plane for Memoria. It is the shared state machine across profiles and sessions. Every long-lived piece of work lives on the board until a human approves it into the vault.

This document covers the state machine, lane structure, board schema, review gate, retries, comments, and handoff patterns.

## Why the board is the orchestration layer

Three things would otherwise be implicit and fragile:

- **Task persistence across sessions.** Without a board, in-flight work lives only in chat history, which is lossy and not queryable.
- **Retries and history.** Without a card identity that survives failures, every retry becomes a new task with no memory of the previous attempt.
- **Handoffs between profiles.** Without a structured handoff note, the next worker has to re-derive context from the conversation.

The board solves all three by being the durable identity of in-flight work.

## State machine

```text
pending ──► ready ──► active ──► awaiting-review ──► approved ──► done
                          │             │                              ▲
                          │             └────────► rejected ───────────┘
                          │                            │   (outcome: discarded — abandon)
                          │                            │   (outcome: superseded — spawn new pending card)
                          │
                          ├── retry-needed ──► active    (transient failure; auto-retry)
                          └── blocked-on-human            (orthogonal; human input)
```

| State | Meaning |
| --- | --- |
| `pending` | Card created but specification still in progress. Operator-only state; dispatcher ignores. Transition to `ready` is a deliberate human action. (Named `pending` to avoid collision with the `draft` note-status used by code-notes and workbench drafts — see [schema.md](reference/schema.md).) |
| `ready` | Specified and dispatchable. The dispatcher will claim this card when a matching-lane worker is available. |
| `active` | A profile owns the card and is executing. |
| `blocked-on-human` | Needs a human decision the worker cannot make. Parallels future `blocked-on-dep` if inter-card dependencies are added. |
| `awaiting-review` | Worker finished; operator must approve (Verifier or Linter may have already produced a recommendation). |
| `rejected` | Operator declined the work this review pass. From here the operator chooses what comes next: spawn a revision card with new specification (original closes with `outcome: superseded`) or abandon entirely (closes with `outcome: discarded`). The original card never reopens. |
| `retry-needed` | Recoverable execution failure (tool error, API timeout); the dispatcher will re-attempt on the same card. |
| `approved` | Operator accepted the work. |
| `done` | Canonical, archived, or shipped. Terminal state. |

The card stays on the board until it reaches a terminal state.

## Why every state matters

- **`pending` vs `ready`** — separates specification from dispatch. A card in `pending` may be missing `lane`, `canonical_target`, or a coherent task description. Promotion to `ready` is an explicit "I'm done thinking about scope" moment that the dispatcher waits for. The closest Hermes-default equivalent is the Triage state, but driven by operator action rather than an orchestrator — see [Cards born in `pending` vs cards born in `ready`](#cards-born-in-pending-vs-cards-born-in-ready) below.
- **`ready` vs `active`** — separates dispatch from execution. Multiple cards can be `ready`; only one profile holds `active` at a time per card.
- **`blocked-on-human` vs `awaiting-review`** — both block, but for different reasons. `blocked-on-human` is a decision the worker cannot make at all (the work itself can't proceed); `awaiting-review` is a check on already-completed work.
- **`rejected` vs `retry-needed`** — `rejected` is a quality decision (the work was wrong); `retry-needed` is an execution problem (a tool failed, a query timed out). `rejected` is operator-driven and exits via discard-or-supersede; `retry-needed` is dispatcher-driven and re-attempts the same card.
- **`approved` vs `done`** — separation matters when promotion involves moving files or running follow-on tasks. `approved` says "the work is good"; `done` says "everything downstream has been handled."

The key discipline: cards never close on a worker's say-so. The card lives until the operator changes the review state.

## Cards born in `pending` vs cards born in `ready`

The default for **operator-created cards** is `pending`. The command-palette commands that create cards (`Memoria: new project`, `Memoria: ingest source` when invoked manually, etc.) write the card with `status: pending` so the operator can review the auto-populated fields and adjust before dispatch.

The default for **cron-created cards and watcher-triggered cards** is `ready`. These cards are pre-specified by their cron config or trigger rule — there is no half-formed state to recover from, and the design intent is fire-and-forget. The nightly hygiene sweep, the weekly drift detector, file-system watchers on `library.bib`, and git-hook-fired verify cards all skip `pending` entirely.

The rule: **if the card is fully specified at creation time, start at `ready`; if a human still needs to shape it, start at `pending`.** The dispatcher ignores `pending` cards regardless of how they got there, so the distinction is visible only in the operator's UI.

## Post-rejection paths

A card in `rejected` is not "back to the worker." The operator's judgment is "no, this work isn't right." What happens next is a separate decision made by the operator with full context:

| Path | What happens | Outcome marker |
| --- | --- | --- |
| **Supersede** | Operator spawns a new card on the same lane with revised specification — addressing the issues that caused rejection. The new card carries a `supersedes: <original-card-id>` provenance field; the original card closes. This is the canonical "revise and retry" pattern. | Original card → `done` with `outcome: superseded`. New card starts in `pending`. |
| **Discard** | Operator decides the work shouldn't exist. The card closes without a successor. The rejection itself is the answer. | Card → `done` with `outcome: discarded`. |

Both paths are explicit operator actions. There is no implicit "return to lane queue" — every rework starts as a new card with new specs, which is more honest about how revisions actually work (the original prompt was usually wrong, not just the work product).

This is *not* the same as `retry-needed`. Retry-needed is automatic re-dispatch after a transient failure on the same card with the same task packet. Rejection is operator judgment that the task itself needs to be re-specified, which is what a new card is for.

## Worker lanes

Lanes are specialist execution paths under the board. Each lane has a primary profile that owns it.

| Lane | Primary profile | Input | Exit state |
| --- | --- | --- | --- |
| Research | Researcher | Candidate paper / item | `awaiting-review` |
| Cartography | Cartographer | Project brief | `awaiting-review` (scope-map ready) |
| Socratic | Socratic | Operator-initiated | N/A (synchronous; no queue) |
| Writer | Writer | Approved evidence | `awaiting-review` |
| Verify | Verifier | Draft commit | `verify-clean`, `verify-needs-revision`, or `verify-needs-attention` |
| Linter | Linter | Candidate or draft | Pass or fix-needed |
| Coder | Coder | Project brief | `awaiting-review` |

Lanes are execution paths, not separate boards. Every card lives in one board state at a time; the lane determines who can claim it.

There is no Review lane and no Orchestration lane. Approval decisions are operator actions on a card's `status` field, gated by the policy MCP — there's no profile that owns them. Routing is encoded in lane-overrides and Kanban dispatch rules, not delegated to a reasoning agent. See [02-profiles.md](02-profiles.md#routing-without-an-orchestrator).

## Board schema fields

These are the recommended fields on every card. They are the API between the workers, the Verifier (when applicable), and the operator.

| Field | Type | Purpose |
| --- | --- | --- |
| `status` | enum | Current board state (one of the nine above). |
| `assignee` | string | Profile currently holding the card. |
| `lane` | enum | Which lane the card belongs to. |
| `blocked_reason` | text | Why the card is blocked, when applicable. |
| `retry_count` | int | Number of recoverable failures so far. |
| `handoff_note` | text | Human-readable handoff summary (Blocker / Tried / Next). |
| `task_packet` | json | Structured [task packet](reference/glossary.md#board-and-cards) for the next worker. Self-contained — receiving worker needs nothing else to start. See [Handoff pattern](#handoff-pattern). |
| `last_updated` | timestamp | When state last changed. |
| `canonical_target` | path | Where the output should land if approved (e.g. `30-synthesis/01-permanent/xyz.md`). |
| `review_state` | enum | Independent of `status`: `unreviewed`, `requested`, `in-review`, `approved`, `rejected`. |
| `review_owner` | string | Who owes the next review decision. |
| `review_requested_at` | timestamp | When the worker handed off for review. |
| `reviewed_at` | timestamp | When the operator (or Verifier for the agent-recommendation phase) acted on the review. |

`review_state` is separate from `status` deliberately. A card can be `active` while `review_state` is still `unreviewed`. This lets dashboards and dispatch logic query review independently.

## Entity vocabulary

The board fields above implicitly track five distinct entities. Naming them is useful because queries, dashboards, and the audit log all touch the same vocabulary — and conflating them produces confusing reports ("how many handoffs happened this week?" is unanswerable if handoffs live inside a free-text field).

| Entity | What it is | Where it lives on the card |
| --- | --- | --- |
| **Task** | A unit of work. One card = one task. | The card itself; identified by `task_id`. |
| **Handoff** | An event where one profile passes work to another. Multiple handoffs per task are normal (Researcher creates source → operator triages → Writer drafts → Verifier traces). | The `task_packet` field carries the *current* handoff; comment log records the history. |
| **Artifact** | An output produced by the task — a source note, a synthesis draft, a code module, a deliverable. | The `canonical_target` field points to the artifact path; multi-artifact tasks list paths in `task_packet.expected_outputs`. |
| **Verdict** | A review decision on the task: `approve`, `reject`, or `escalate` (see the verdict vocabulary below). Issued by the operator (always for `status: approved`) or by Verifier for the agent-recommendation phase of a verify card. One per review pass. Routing after a `reject` verdict (discard vs supersede) is a separate operator action, not part of the verdict itself. | The `review_state` field carries the latest verdict; comment log records prior verdicts. |
| **State transition** | A change in `status` (`ready → active`, `active → awaiting-review`, etc.). Multiple transitions per task. | Implicit in card history; `last_updated` records the most recent transition timestamp. |

### Why not a separate registry

The doc that surfaced this vocabulary (`memoria_task_registry_schema.md` in `raw/`) proposes a separate SQLite registry with one table per entity. Memoria does not adopt that — the Hermes native Kanban already holds task and transition state, and a parallel registry creates two sources of truth that have to be kept in sync.

Instead, the entities above are **projections of card state**, not separate stores:

- **Aggregate views** for the fleet-observability dashboard come from a scheduled aggregator that reads card history and writes `lane-metric` and `skill-metric` notes to `00-meta/08-metrics/`.
- **Audit trail** for individual actions comes from `00-meta/04-logs/audit.jsonl` (written by the policy MCP).
- **Card detail** (task + current handoff + current verdict) lives on the card itself.

This keeps the board as the single source of truth while still giving the design clear vocabulary for what the card tracks.

## The review gate

Review is a first-class state, not a comment. The full rules:

### Rule 0: Agent recommendation and human approval are separate

A card has two distinct approval signals tracked in two different fields:

- **`review_state: approved`** — an agent (typically Verifier for drafts, Linter for structural checks) has examined the work and judged it complete. This is the *agent-recommended* state. There is no Reviewer profile; verdicts come from the profile whose job included the check.
- **`status: approved`** — the operator has accepted the work as canonical. Moving `status` to `approved` is always a human action.

The two are not equivalent. A card with `review_state: approved` and `status: awaiting-review` means "agent recommends, awaiting human sign-off." A card with `status: approved` and `review_state: unreviewed` is a configuration bug — humans should not advance work past agent checks silently.

The full pipeline (with both signals visible):

```text
ready ──► active ──► awaiting-review ──► review_state: approved ──► status: approved ──► done
                                              (agent gate)              (human gate)
```

The agent gate is necessary but not sufficient. The human approval is the canonical promotion gate; the agent verdict is the recommendation feeding into it.

### Rule 1: Review is a state, not a comment

A card only reaches `status: approved` after a human changes its state, not because a worker says it is finished. The required fields:

- `review_state` — set by an agent (Verifier, Linter) when a recommendation is produced, or by the operator on direct approval
- `review_owner` — who owes the next review decision
- `review_requested_at`
- `reviewed_at`
- `blocked_reason` (when applicable)

If a comment says "I reviewed this" but `review_state` is still `unreviewed`, the card is unreviewed. The state field is authoritative.

### Rule 2: Review states block dispatch

Cards in `awaiting-review`, `blocked-on-human`, or `pending` should not be claimable by non-review workers until the state changes. The dispatch logic queries `review_state` directly. Non-dispatchable means non-dispatchable.

### Rule 3: Review has ownership

The reviewing party (operator, Verifier, Linter) is explicitly recorded in `review_owner`, so the board can show who owes the next decision. No silent promotes. Accountability is preserved.

### Rule 4: Review is a gate in the lifecycle

Writer and coder can finish their own slices, but the card remains live until human approval changes the state to `approved`. If the operator instead rejects, the card closes with an explicit outcome — `superseded` if a revision card spawns, `discarded` if the work is abandoned. The original card never silently reopens; revisions live on a new card with provenance back to the original (see [Post-rejection paths](#post-rejection-paths)).

### Review verdict vocabulary

Every review pass ends with exactly one of three verdicts. The verdict is recorded as a comment on the card and drives the next state transition. **The operator issues the verdict** (the canonical approval); agents (Verifier, Linter) may attach recommendation-only verdicts that the operator can adopt or override.

| Verdict | Meaning | Resulting state |
| --- | --- | --- |
| `approve` | The work meets standards; advance to canonical. | `approved` → `done` |
| `reject` | The work is not right as it stands. The operator separately decides whether to spawn a revision card (with new specification) or discard the work entirely. | `rejected` |
| `escalate` | The decision exceeds the reviewer's scope; flag for a separate decision (typically rewriting the task packet or the lane-override). | `blocked-on-human` |

Rules:

- Every verdict must include a short reason and, when relevant, the exact note or field that needs correction.
- "Looks fine" is not a verdict. The reviewer must choose one of the three.
- The verdict and reason live on the card; do not rely on chat or external comments.
- `escalate` is the right verdict whenever uncertainty would otherwise default to `approve`.

**Verifier-specific verdicts.** Verifier produces a more granular verdict triple on `verify` cards: `verify-clean`, `verify-needs-revision`, `verify-needs-attention` (see [verifier.md](profiles/verifier.md#verdict-semantics)). These are recommendations the operator translates into one of the three canonical verdicts above. A `verify-clean` is typically promoted to `approve`; `verify-needs-revision` to `reject` (the operator then chooses to spawn a revision card or discard, per [Post-rejection paths](#post-rejection-paths)); `verify-needs-attention` to either `reject` or `escalate` depending on the operator's read.

## Persistence pattern

The board is where work memory lives across sessions and across worker handoffs.

- **Same card, same identity.** A retry does not create a new card.
- **Session-safe memory.** Closing your chat does not lose the work; the card persists.
- **No chat-history dependence.** The next worker reads the card, not the transcript.
- **Visible until closed.** A card in any non-terminal state is on the board.
- **Shared memory across roles.** The handoff note is the API between profiles.

## Retry pattern

Failed work moves to `retry-needed` instead of disappearing. The next attempt happens on the same card.

The retry comment should capture:

- **Failure reason.** What broke (tool error, API timeout, unclear input).
- **What was attempted.** Which approaches the previous worker tried.
- **Next action.** The exact thing the next claim should try.

`retry_count` increments. The same or a different profile can reclaim.

### Escalation threshold

**Default: `retry_count > 3` auto-moves the card to `blocked-on-human` with `blocked_reason: "retry_threshold_exceeded"`.** Three attempts is the operating point — enough that transient failures (a rate-limited API, a flaky external lookup) resolve themselves on retry, few enough that a structurally broken task doesn't burn the lane on the same failure mode all night. The threshold is configurable per lane in the lane-override file for lanes where the cost profile justifies a different number — the research lane may tolerate `>5` because each retry is cheap, the writer lane may want `>2` because each retry is a model call. The Kanban dispatcher reads the threshold from the lane-override; no profile is "responsible" for enforcing it.

The card stays in `blocked-on-human` until the operator either revises the task packet (and resets `retry_count` to 0), promotes the issue out of band (rewriting the prompt, fixing a tool), or closes the card with `blocked_reason: "infeasible"`. The retry counter never auto-decrements; only an operator decision resets it.

This avoids three problems: duplicate cards for the same work, lost history about why a task is hard, and the temptation to give up silently. The escalation threshold adds a fourth guard: a brittle prompt or a broken tool can't quietly burn through API budget overnight.

## Handoff pattern

Handoffs carry two things: a **human-readable summary** for board readers, and a **structured task packet** for the next worker (and the policy MCP). Both live on the card.

### Human-readable summary

The `handoff_note` field is short prose with three lines. It is what the next worker or the operator sees at a glance:

```text
Blocker: [what's stopping forward progress]
Tried: [what's been attempted; what was learned]
Next: [the exact action the next worker should take]
```

When the operator spawns a revision card after a rejection, the new card's handoff note describes what was wrong with the original (the new card carries `supersedes: <original-id>` for provenance; the original closes with `outcome: superseded`). When a Researcher hands off to Verifier for filing-time similarity check, the same handoff format applies. When a Writer's draft fires the Verifier hook, the same.

### Structured task packet

The `task_packet` field is JSON. It is what the next worker (or a delegated child agent) consumes programmatically. Hermes delegated children return summaries rather than sharing live parent state, so every handoff packet must be **self-contained** — the receiving worker should need nothing beyond the packet to start work.

```json
{
  "task_id": "TASK-2026-05-25-014",
  "origin_profile": "operator",
  "target_profile": "memoria-researcher",
  "goal": "Find recent systematic reviews on persuasive digital health interventions",
  "context": {
    "research_direction": "digital health coaching",
    "project": "memoria-health-coaching"
  },
  "allowed_paths": [
    "10-inbox/03-candidates/**",
    "20-sources/01-literature/**"
  ],
  "expected_outputs": [
    "candidate source notes",
    "triage proposal",
    "handoff note"
  ],
  "review_checks": [
    "stable identifier present",
    "draft classification included"
  ],
  "next_state": "awaiting-review"
}
```

Field meanings:

- `task_id` — links back to the card. Required.
- `origin_profile` / `target_profile` — who handed off, who receives. Required.
- `goal` — one sentence describing the outcome the receiving worker is responsible for.
- `context` — structured key/value pairs the worker needs (project, research_direction, related cards). Not free prose.
- `allowed_paths` — the worker's write scope for this card specifically. The policy MCP cross-checks against the lane override; the packet can narrow but never widen.
- `expected_outputs` — what the receiving worker must produce before the card can exit. Verifier or Linter checks these where applicable.
- `review_checks` — what the agent (Verifier, Linter) or the operator will verify before approving.
- `next_state` — the exit state the worker should move the card to when complete.

The `handoff_note` and `task_packet` together form the durable trail. The conversation does not.

### Why both forms

The prose `handoff_note` is for humans scanning the board; the structured `task_packet` is for the next worker (and tools that read the card). Trying to use one for both produces packets too verbose to scan and prose too vague to act on programmatically. Keep them separate.

## Cross-role flow

A typical card moves through:

```text
Trigger creates card ──► Specialist executes ──► Verifier / Linter recommendation ──► Operator approves
                                                                                            │
                                                                                            ├─ approved ──► Done
                                                                                            │
                                                                                            └─ rejected ──► Done (outcome: superseded or discarded)
                                                                                                                  │
                                                                                                                  └─ (optional) new pending card with provenance
```

Triggers (operator action, cron, git hook, file watcher) create cards according to the lane-override rules. Specialists act within their lane. Verifier produces a verification verdict on `verify` cards; Linter produces structural reports. The operator approves or blocks.

## Lane-level rules

Each lane has its own exit-state contract.

| Lane | Default exit | What "exit" means |
| --- | --- | --- |
| Research | `awaiting-review` | Source notes created, enriched, classified; ready for operator triage. |
| Cartography | `awaiting-review` | Scope-map (or gap-report, cluster-map, comparative-brief) written; ready for operator decision. |
| Socratic | N/A | Synchronous; no card lifecycle. Operator closes the ACP pane to end. |
| Writer | `awaiting-review` | Synthesis draft ready; commit triggers Verifier hook for `verify` cards. |
| Verify | `verify-clean` / `verify-needs-revision` / `verify-needs-attention` | Verification report written; operator translates to canonical verdict. |
| Linter | Pass or fix-needed | Structural check completed; report attached as comment. |
| Coder | `awaiting-review` | Code artifact ready; needs review of code and provenance. |

The exit states are how the board enforces that nobody self-approves.

## WIP limits

Recommended work-in-progress limits to prevent overload:

- **Active per profile**: 1. A profile holds one `active` card at a time.
- **Review queue depth**: bounded (e.g., 5). When the operator's `awaiting-review` queue exceeds the limit, the Kanban dispatcher delays new card creation on that lane (or escalates a notification via Telegram, depending on configuration).
- **Synthesis lane**: bounded (e.g., 3). Too many synthesis cards at once means quality drops because evidence cannot be fully integrated.

These are operational tuning parameters, not architectural constants.

## Board implementation: Hermes native Kanban

Memoria uses the **Hermes native Kanban board**. This is canonical — not one option among several.

- [Hermes Kanban feature](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Kanban tutorial](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial)
- [Worker lanes](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes)

Why Hermes native:

- The whole architecture assumes Hermes profiles; the native board gives profiles first-class lane semantics without bridging.
- Worker-lane contracts are built in — `delegate_task` for one-off children, profile binding for persistent lanes.
- Review states are native fields, not extensions of a generic board schema.
- Cross-session persistence and retry semantics match what's specified in the schema fields above.

### Dispatch interval

The Hermes Kanban dispatcher polls the queue every **60 seconds** (`dispatch_in_gateway: true`, `dispatch_interval_seconds: 60`). This is the canonical Memoria setting, not a default to be retuned per deployment.

- **Faster (e.g., 10s)** wastes resources — most cards aren't dispatchable on most loops; the polling overhead grows linearly with no corresponding throughput gain.
- **Slower (e.g., 5 min)** makes the system feel laggy — the operator triggers a workflow from the command palette and waits visibly long enough to wonder if it landed.

The 60-second cadence is also what makes the async / Kanban surface feel ambient: dropped PDFs get picked up within a minute (fast enough that the operator forgets they triggered it); cron-created cards reach their lanes promptly without consuming idle cycles. Don't change it without a specific reason; if a specific lane needs faster response, push the priority of its cards rather than tightening the global interval.

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
- **Not a place for canonical claims.** Claims live in `30-synthesis/01-permanent/`. The board references them; it doesn't hold them.
- **Not a substitute for review.** A card with `review_state: approved` is meaningful; a card with no review state set is not.

## Anti-patterns

- **Closing cards on worker say-so.** Always wait for the review state to change.
- **Creating a new card for a retry.** Reuse the existing card; increment `retry_count`.
- **Burying review status in comments.** If `review_state` is the source of truth, queries must use it; if comments are the source of truth, review is implicit and unenforceable.
- **Unbounded lanes.** Without WIP limits, the synthesis or review queue fills up and the human becomes the bottleneck silently.

## Next

- For how the board interacts with workflows (the stage-gated pipeline): [04-workflows.md](04-workflows.md).
- For the per-profile contracts that interpret these states: [02-profiles.md](02-profiles.md).
- For how dashboards visualize board state: [06-surfaces.md](06-surfaces.md).
