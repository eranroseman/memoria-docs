---
mode: reference
audience: operator
topic: board
---

# Board states and the review gate

The board tracks **two orthogonal lifecycles**:

1. **Execution** — the Hermes built-in `status` field, a fixed seven-value enum. This is what the dispatcher and workers move.
2. **Review** — a Memoria overlay stored in `metadata.review_status` (plus `metadata.agent_verdict`). Hermes has no review gate, so Memoria layers one on. It rides on top of `status: done`.

Keeping them separate is what lets `status` hold real Hermes values while Memoria's approval semantics live where Hermes has a slot for them (the free-form `metadata` JSON). For the field definitions see [card-schema.md](card-schema.md); for the conceptual narrative see [README.md](README.md); for term disambiguation see the [glossary](../glossary.md).

## Execution lifecycle (Hermes `status`)

```text
triage ──► todo ──► ready ──► running ──► done ──► archived
                      ▲          │
          (retry) ────┘          └──► blocked ──(unblock)──► ready
```

| `status` | Meaning | Moved by |
| --- | --- | --- |
| `triage` | Card created; specification still in progress. The dispatcher ignores it until specified. | `hermes kanban specify` (flesh out a triage task into a concrete spec) or `decompose` (fan out into child tasks). |
| `todo` | Specified and on the backlog, not yet released for dispatch. | Human — releases the spec to `ready`. There is no orchestrator ([routing lives in lane-overrides + dispatch rules](../profiles/README.md#routing-without-an-orchestrator)). |
| `ready` | Dispatchable. The dispatcher will hand this to a matching-lane worker. | `hermes kanban dispatch` runs one dispatcher pass. |
| `running` | A profile owns the card and is executing. | The **dispatcher** — it atomically claims a `ready` card and spawns the assigned profile. Workers do **not** self-claim; `hermes kanban claim` exists for manual/script use. |
| `blocked` | Needs a human decision the worker cannot make; carries a `reason`. | Worker sets it via `kanban_block`; the human clears it with `hermes kanban unblock` → `ready`. |
| `done` | Worker finished. In Memoria this is also where the **review overlay** applies — a `done` card is not canonical until reviewed. | `kanban_complete` (with `summary` + `metadata`). |
| `archived` | Terminal. Canonical and shipped, or abandoned. | `hermes kanban archive`. |

Notes:

- **Retries** are not a distinct status. A recoverable run failure (`outcome: crashed` or `gave_up`, within `max_retries`) returns the card to `ready` for re-dispatch.
- Board `status` values and note-vocabulary values are disjoint — Hermes `triage` does not collide with the note `draft` type — so the board needs no special state name to avoid overlap. See [frontmatter-schema.md](../vault/frontmatter-schema.md).

## Review lifecycle (Memoria overlay, `metadata.review_status`)

```text
unreviewed ──► requested ──► in-review ──► approved   (canonical; then archived)
                                       └─► rejected   (then archived = discarded, or new card = superseded)
```

| `review_status` | Meaning |
| --- | --- |
| `unreviewed` | No review requested yet. The default while a card is `triage` → `running`. |
| `requested` | Worker handed off for review (set on `kanban_complete`; `status` is now `done`). |
| `in-review` | A human (or agent, for its recommendation) is examining the work. |
| `approved` | The human accepted the work as canonical. |
| `rejected` | The human declined this pass. |

The review lifecycle only becomes meaningful once `status` reaches `done`. A card can be `status: running` with `review_status: unreviewed` — that is the normal mid-flight state.

## Worker lanes

Lanes are specialist execution paths under the board. A lane **is** an `assignee` value — the dispatcher routes a card to a lane by matching `task.assignee` to a Hermes profile (or a registered external worker). There is no separate `lane` field; tasks with an unresolved `assignee` stay in `ready` with a `skipped_nonspawnable` event (a Hermes dispatch event meaning no worker matched the `assignee`).

Every lane exits to `status: done`; what differs is what "done" means and which review signal applies. The exit contract is how the board enforces that nobody self-approves — reaching `done` is not the same as `review_status: approved`.

| Profile (lane `assignee`) | Input | Exit | What "done" means |
| --- | --- | --- | --- |
| Librarian | Candidate paper / item | `done` (`review_status: requested`) | Paper notes created, enriched, classified; ready for human classification. |
| Mapper | Project brief | `done` (`review_status: requested`) | Scope-map (or gap-report, cluster-map, comparative-brief) written; ready for human decision. |
| Writer | Approved evidence | `done` (`review_status: requested`) | Answer draft ready; the commit fires the Verifier hook that creates a `verify` card. |
| Verifier | Draft commit | `done` (`agent_verdict` = the `verify-*` triple) | Verification report written; the human translates `verify-clean` / `verify-needs-revision` / `verify-needs-attention` into a verdict. |
| Linter | Candidate or draft | `done` (`agent_verdict`: `approve` / `reject` / `escalate`) | Structural check completed (pass or fix-needed); report attached as a comment. |
| Coder | Project brief | `done` (`review_status: requested`) | Code artifact ready; needs review of code and provenance. |

**Socratic is not a board lane.** It runs synchronously through the ACP pane — the human opens it, converses, and closes it. It never appears on the board, claims a card, or produces a `done` card; it has no card lifecycle. (It's a profile, just not a board-dispatched one.)

There is no Review lane and no Orchestration lane either. Approval is a human action on `metadata.review_status`, gated by the policy MCP — no profile owns it; routing is encoded in lane-overrides and Kanban dispatch rules, not delegated to a reasoning agent. For the rationale, see [profiles/README.md](../profiles/README.md#routing-without-an-orchestrator) and the README's [Why no Reviewer and no Orchestrator](README.md#why-no-reviewer-and-no-orchestrator).

## The review gate

Review is first-class state in `metadata`, not a comment. The full rules:

### Rule 1: Agent recommendation and human acceptance are separate

A card carries two distinct approval signals:

- **`metadata.agent_verdict: approve`** — an agent (typically Verifier for drafts, Linter for structural checks) examined the work and judged it complete. This is the *agent recommendation*. There is no Reviewer profile; verdicts come from the profile whose job included the check.
- **`metadata.review_status: approved`** — the human accepted the work as canonical. This is always a human action.

The two are not equivalent. A card with `agent_verdict: approve` and `review_status: requested` means "agent recommends, awaiting human sign-off." A card promoted to canonical with `agent_verdict` unset, when a check was expected, is a configuration bug — humans should not advance work past agent checks silently.

The full pipeline (both signals visible, all riding on `status: done`):

```text
status: running ──► done ─────────────────────────────────► archived
agent:                 agent_verdict: approve   (agent gate)
review:                review_status: requested ──► approved (human gate)
```

The agent gate is necessary but not sufficient. Human acceptance (`review_status: approved`) is the canonical promotion gate; the agent verdict is the recommendation feeding into it.

### Rule 2: Review is a state, not a comment

A card is canonical only after a human sets `review_status: approved`, not because a worker says it is finished. The review fields all live in the `metadata` overlay — the full list is in [card-schema.md → Memoria overlay fields](card-schema.md#memoria-overlay-fields-inside-metadata). If a comment says "I reviewed this" but `review_status` is still `unreviewed`, the card is unreviewed. The field is authoritative.

### Rule 3: Review states block dispatch

Cards in `done` (awaiting review), `blocked`, `triage`, or `todo` are not claimable by non-review workers until state changes. Hermes does not dispatch these statuses; Memoria additionally checks `review_status` before promoting. Non-dispatchable means non-dispatchable.

### Rule 4: Review has ownership

The reviewing party (human, Verifier, Linter) is explicitly recorded in `review_owner`, so the board can show who owes the next decision. No silent promotes. Accountability is preserved.

### Rule 5: Review is a gate in the lifecycle

Writer and Coder can finish their own slices (`status: done`), but the card is not canonical until a human sets `review_status: approved`. On rejection the original card never quietly reopens — it is archived with a stated outcome, and any rework starts on a fresh card. The supersede-vs-discard decision is a separate human action; see [Post-rejection paths](README.md#post-rejection-paths).

### Review verdict vocabulary

These three verdicts are the **human's** decision set — the choice made at the review gate. Agents never issue them directly: Verifier and Linter attach *recommendations* in `metadata.agent_verdict` (Verifier's are the finer-grained `verify-*` triple, below) that the human adopts or overrides. Every review pass ends with exactly one of the three, and it drives the next transition.

| Verdict | Meaning | Resulting state |
| --- | --- | --- |
| `approve` | The work meets standards; advance to canonical. | `review_status: approved` → `status: archived` |
| `reject` | The work is not right as it stands. The human separately decides whether to spawn a revision card or discard. | `review_status: rejected` |
| `escalate` | The decision exceeds the reviewer's scope; flag for a separate decision (typically rewriting the handoff payload or the lane-override). | `status: blocked` |

Rules:

- Every verdict must include a short reason and, when relevant, the exact note or field that needs correction.
- "Looks fine" is not a verdict. The reviewer must choose one of the three.
- The verdict and reason live on the card (`metadata` + comment log); do not rely on chat or external comments.
- `escalate` is the right verdict whenever uncertainty would otherwise default to `approve`.

**Verifier-specific verdicts.** Verifier produces a more granular triple in `metadata.agent_verdict` on `verify` cards: `verify-clean`, `verify-needs-revision`, `verify-needs-attention` (see [verifier.md](../profiles/verifier.md#verdict-semantics)). These are recommendations the human translates into one of the three verdicts. A `verify-clean` is typically promoted to `approve`; `verify-needs-revision` to `reject` (the human then chooses to spawn a revision card or discard, per [Post-rejection paths](README.md#post-rejection-paths)); `verify-needs-attention` to either `reject` or `escalate` depending on the human's read.

## WIP limits

Recommended work-in-progress limits to prevent overload:

- **Active per profile**: 1. A profile holds one `running` card at a time.
- **Review queue depth**: bounded (e.g. 5). When the human's queue of `done` cards awaiting review exceeds the limit, the Kanban dispatcher delays new card creation on that lane (or escalates a notification via Telegram, depending on configuration).
- **Writer lane (synthesis)**: bounded (e.g. 3). Too many drafts in flight at once means quality drops because evidence cannot be fully integrated.

These are operational tuning parameters, not architectural constants.

**Configuration and enforcement.** *Active-per-profile = 1* is enforced natively by Hermes — a profile holds one `running` card at a time. The *review-queue* and *synthesis* caps are Memoria-side policies the dispatcher applies before it creates or releases new cards on a lane: when a lane is at its cap, new-card creation on that lane is delayed (or a Telegram notification is escalated, per configuration). They are per-lane tuning values, not Hermes built-ins.
