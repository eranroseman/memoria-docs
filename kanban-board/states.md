---
mode: reference
audience: operator
topic: board
---

# Board states and review gate (reference)

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
| `todo` | Specified and on the backlog, not yet released for dispatch. | Human/orchestrator. |
| `ready` | Dispatchable. The dispatcher will hand this to a matching-lane worker. | `hermes kanban dispatch`; a worker uses `claim`. |
| `running` | A profile owns the card and is executing. | `kanban_claim` / `hermes kanban claim`. |
| `blocked` | Needs a human decision the worker cannot make; carries a `reason`. | `kanban_block`; cleared with `kanban_unblock` → `ready`. |
| `done` | Worker finished. In Memoria this is also where the **review overlay** applies — a `done` card is not canonical until reviewed. | `kanban_complete` (with `summary` + `metadata`). |
| `archived` | Terminal. Canonical and shipped, or abandoned. | `hermes kanban archive`. |

Notes:

- **Retries** are not a distinct status. A recoverable run failure (`outcome: crashed` or `gave_up`, within `max_retries`) returns the card to `ready` for re-dispatch. The old `requeued` state is just this.
- Hermes `triage` is naturally disjoint from the note `draft` type, so the old concern about board `status` colliding with note vocabulary no longer needs a special state name — `triage` carries no overlap. See [frontmatter-schema.md](../vault/frontmatter-schema.md).

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

## Crosswalk: old Memoria states

Earlier drafts modeled a single nine-value `status` enum. Those values map onto the two lifecycles as follows. Docs that still use the old names should be read through this table until they are updated.

| Old Memoria state | `status` | `metadata.review_status` | Notes |
| --- | --- | --- | --- |
| `unspecified` | `triage` | — | Needs `hermes kanban specify`. |
| `queued` | `ready` | `unreviewed` | Dispatchable; `todo` is the specified-but-staged precursor. |
| `claimed` | `running` | `unreviewed` | A profile is executing. |
| `blocked-on-human` | `blocked` | — | `reason` set via `kanban_block`. |
| `requeued` | `ready` | `unreviewed` | Re-dispatched after a recoverable run failure, within `max_retries`. |
| `delivered` | `done` | `requested` | Worker finished; awaiting human review. |
| `accepted` | `done` | `approved` | Human accepted as canonical. |
| `declined` | `done` | `rejected` | Then archived (discarded) or superseded by a new card. |
| `closed` | `archived` | (terminal) | Canonical and shipped, or abandoned. |

The card stays on the board until `status` reaches `archived`.

## Worker lanes

Lanes are specialist execution paths under the board. A lane **is** an `assignee` value — the dispatcher routes a card to a lane by matching `task.assignee` to a Hermes profile (or a registered external worker). There is no separate `lane` field; tasks with an unresolved `assignee` stay in `ready` with a `skipped_nonspawnable` event.

Every lane exits to `status: done` (Socratic excepted — it is synchronous, with no card lifecycle); what differs is what "done" means and which review signal applies. The exit contract is how the board enforces that nobody self-approves — reaching `done` is not the same as `review_status: approved`.

| Lane (`assignee`) | Primary profile | Input | Exit | What "done" means |
| --- | --- | --- | --- | --- |
| Library | Librarian | Candidate paper / item | `done` (`review_status: requested`) | Paper notes created, enriched, classified; ready for human classification. |
| Mapping | Mapper | Project brief | `done` (`review_status: requested`) | Scope-map (or gap-report, cluster-map, comparative-brief) written; ready for human decision. |
| Socratic | Socratic | Human-initiated | N/A — synchronous; no card lifecycle | Human closes the ACP pane to end. |
| Writer | Writer | Approved evidence | `done` (`review_status: requested`) | Answer draft ready; commit triggers the Verifier hook for `verify` cards. |
| Verify | Verifier | Draft commit | `done` (`agent_verdict` triple) | Verification report written; human translates `verify-clean` / `verify-needs-revision` / `verify-needs-attention` to a verdict. |
| Linter | Linter | Candidate or draft | `done` (`agent_verdict`) | Structural check completed (pass or fix-needed); report attached as comment. |
| Coder | Coder | Project brief | `done` (`review_status: requested`) | Code artifact ready; needs review of code and provenance. |

Lanes are execution paths, not separate boards. Every card sits in one `status` at a time; the `assignee` determines who can claim it.

There is no Review lane and no Orchestration lane. Approval decisions are human actions on `metadata.review_status`, gated by the policy MCP — there's no profile that owns them. Routing is encoded in lane-overrides and Kanban dispatch rules, not delegated to a reasoning agent. See [profiles/README.md](../profiles/README.md#routing-without-an-orchestrator).

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

A card is canonical only after a human sets `review_status: approved`, not because a worker says it is finished. The required `metadata` keys:

- `review_status` — set by the human on the decision; `requested` by the worker on handoff
- `agent_verdict` — set by an agent (Verifier, Linter) when a recommendation is produced
- `review_owner` — who owes the next review decision
- `review_requested_at`
- `reviewed_at`
- `reason` — when the card is `blocked`

If a comment says "I reviewed this" but `review_status` is still `unreviewed`, the card is unreviewed. The field is authoritative.

### Rule 3: Review states block dispatch

Cards in `done` (awaiting review), `blocked`, `triage`, or `todo` are not claimable by non-review workers until state changes. Hermes does not dispatch these statuses; Memoria additionally checks `review_status` before promoting. Non-dispatchable means non-dispatchable.

### Rule 4: Review has ownership

The reviewing party (human, Verifier, Linter) is explicitly recorded in `review_owner`, so the board can show who owes the next decision. No silent promotes. Accountability is preserved.

### Rule 5: Review is a gate in the lifecycle

Writer and Coder can finish their own slices (`status: done`), but the card is not canonical until a human sets `review_status: approved`. On rejection the original card never silently reopens — it is archived with an explicit outcome, and any rework starts on a fresh card. The supersede-vs-discard decision is a separate human action; see [Post-rejection paths](README.md#post-rejection-paths).

### Review verdict vocabulary

Every review pass ends with exactly one of three verdicts. The verdict drives the next transition. **The human issues the verdict**; agents (Verifier, Linter) attach recommendation-only verdicts in `agent_verdict` that the human can adopt or override.

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
