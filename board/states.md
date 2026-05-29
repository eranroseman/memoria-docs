---
mode: reference
audience: operator
topic: board
---

# Board states and review gate (reference)

The state machine, worker lanes, lane-level exit contracts, the review gate, and verdict vocabulary. For the conceptual narrative of why these states exist, see [README.md](README.md).

```text
unspecified ──► queued ──► claimed ──► delivered ──► accepted ──► closed
                             │             │                          ▲
                             │             └────────► declined ───────┘
                             │                          │   (outcome: discarded — abandon)
                             │                          │   (outcome: superseded — spawn new unspecified card)
                             │
                             ├── requeued ──► claimed    (transient failure; auto-retry)
                             └── blocked-on-human         (orthogonal; human input)
```

| State | Meaning |
| --- | --- |
| `unspecified` | Card created but specification still in progress. Human-only state; dispatcher ignores. Transition to `queued` is a deliberate human action. (Named `unspecified`, not `draft`, to keep board `status` values disjoint from note vocabulary — `draft` is already a note type, with its own `draft_stage` field. See [frontmatter-schema.md](../vault/frontmatter-schema.md).) |
| `queued` | Specified and dispatchable. The dispatcher will claim this card when a matching-lane worker is available. |
| `claimed` | A profile owns the card and is executing. |
| `blocked-on-human` | Needs a human decision the worker cannot make. Parallels future `blocked-on-dep` if inter-card dependencies are added. |
| `delivered` | Worker finished; human must approve (Verifier or Linter may have already produced a recommendation). |
| `declined` | Human declined the work this review pass. From here the human chooses what comes next: spawn a revision card with new specification (original closes with `outcome: superseded`) or abandon entirely (closes with `outcome: discarded`). The original card never reopens. |
| `requeued` | Recoverable execution failure (tool error, API timeout); the dispatcher will re-attempt on the same card. |
| `accepted` | Human accepted the work. |
| `closed` | Canonical, archived, or shipped. Terminal state. |

The card stays on the board until it reaches a terminal state.

## Worker lanes

Lanes are specialist execution paths under the board. Each lane has a primary profile that owns it.

| Lane | Primary profile | Input | Exit state |
| --- | --- | --- | --- |
| Library | Librarian | Candidate paper / item | `delivered` |
| Mapping | Mapper | Project brief | `delivered` (scope-map ready) |
| Socratic | Socratic | Human-initiated | N/A (synchronous; no queue) |
| Writer | Writer | Approved evidence | `delivered` |
| Verify | Verifier | Draft commit | `verify-clean`, `verify-needs-revision`, or `verify-needs-attention` |
| Linter | Linter | Candidate or draft | Pass or fix-needed |
| Coder | Coder | Project brief | `delivered` |

Lanes are execution paths, not separate boards. Every card lives in one board state at a time; the lane determines who can claim it.

There is no Review lane and no Orchestration lane. Approval decisions are human actions on a card's `status` field, gated by the policy MCP — there's no profile that owns them. Routing is encoded in lane-overrides and Kanban dispatch rules, not delegated to a reasoning agent. See [profiles/README.md](../profiles/README.md#routing-without-an-orchestrator).

## The review gate

Review is a first-class state, not a comment. The full rules:

### Rule 0: Agent recommendation and human approval are separate

A card has two distinct approval signals tracked in two different fields:

- **`review_status: approved`** — an agent (typically Verifier for drafts, Linter for structural checks) has examined the work and judged it complete. This is the *agent-recommended* state. There is no Reviewer profile; verdicts come from the profile whose job included the check.
- **`status: accepted`** — the human has accepted the work as canonical. Moving `status` to `accepted` is always a human action.

The two are not equivalent. A card with `review_status: approved` and `status: delivered` means "agent recommends, awaiting human sign-off." A card with `status: accepted` and `review_status: unreviewed` is a configuration bug — humans should not advance work past agent checks silently.

The full pipeline (with both signals visible):

```text
queued ──► claimed ──► delivered ──► review_status: approved ──► status: accepted ──► closed
                                     (agent gate)              (human gate)
```

The agent gate is necessary but not sufficient. The human approval is the canonical promotion gate; the agent verdict is the recommendation feeding into it.

### Rule 1: Review is a state, not a comment

A card only reaches `status: accepted` after a human changes its state, not because a worker says it is finished. The required fields:

- `review_status` — set by an agent (Verifier, Linter) when a recommendation is produced, or by the human on direct approval
- `review_owner` — who owes the next review decision
- `review_requested_at`
- `reviewed_at`
- `blocked_reason` (when applicable)

If a comment says "I reviewed this" but `review_status` is still `unreviewed`, the card is unreviewed. The state field is authoritative.

### Rule 2: Review states block dispatch

Cards in `delivered`, `blocked-on-human`, or `unspecified` should not be claimable by non-review workers until the state changes. The dispatch logic queries `review_status` directly. Non-dispatchable means non-dispatchable.

### Rule 3: Review has ownership

The reviewing party (human, Verifier, Linter) is explicitly recorded in `review_owner`, so the board can show who owes the next decision. No silent promotes. Accountability is preserved.

### Rule 4: Review is a gate in the lifecycle

Writer and Coder can finish their own slices, but the card remains live until human approval changes the state to `accepted`. If the human instead rejects, the card closes with an explicit outcome — `superseded` if a revision card spawns, `discarded` if the work is abandoned. The original card never silently reopens; revisions live on a new card with provenance back to the original (see [Post-rejection paths](README.md#post-rejection-paths)).

### Review verdict vocabulary

Every review pass ends with exactly one of three verdicts. The verdict is recorded as a comment on the card and drives the next state transition. **The human issues the verdict** (the canonical approval); agents (Verifier, Linter) may attach recommendation-only verdicts that the human can adopt or override.

| Verdict | Meaning | Resulting state |
| --- | --- | --- |
| `approve` | The work meets standards; advance to canonical. | `accepted` → `closed` |
| `reject` | The work is not right as it stands. The human separately decides whether to spawn a revision card (with new specification) or discard the work entirely. | `declined` |
| `escalate` | The decision exceeds the reviewer's scope; flag for a separate decision (typically rewriting the task packet or the lane-override). | `blocked-on-human` |

Rules:

- Every verdict must include a short reason and, when relevant, the exact note or field that needs correction.
- "Looks fine" is not a verdict. The reviewer must choose one of the three.
- The verdict and reason live on the card; do not rely on chat or external comments.
- `escalate` is the right verdict whenever uncertainty would otherwise default to `approve`.

**Verifier-specific verdicts.** Verifier produces a more granular verdict triple on `verify` cards: `verify-clean`, `verify-needs-revision`, `verify-needs-attention` (see [verifier.md](../profiles/verifier.md#verdict-semantics)). These are recommendations the human translates into one of the three canonical verdicts above. A `verify-clean` is typically promoted to `approve`; `verify-needs-revision` to `reject` (the human then chooses to spawn a revision card or discard, per [Post-rejection paths](README.md#post-rejection-paths)); `verify-needs-attention` to either `reject` or `escalate` depending on the human's read.

## Lane-level rules

Each lane has its own exit-state contract.

| Lane | Default exit | What "exit" means |
| --- | --- | --- |
| Library | `delivered` | Paper notes created, enriched, classified; ready for human classification. |
| Mapping | `delivered` | Scope-map (or gap-report, cluster-map, comparative-brief) written; ready for human decision. |
| Socratic | N/A | Synchronous; no card lifecycle. Human closes the ACP pane to end. |
| Writer | `delivered` | Answer draft ready; commit triggers Verifier hook for `verify` cards. |
| Verify | `verify-clean` / `verify-needs-revision` / `verify-needs-attention` | Verification report written; human translates to canonical verdict. |
| Linter | Pass or fix-needed | Structural check completed; report attached as comment. |
| Coder | `delivered` | Code artifact ready; needs review of code and provenance. |

The exit states are how the board enforces that nobody self-approves.

## WIP limits

Recommended work-in-progress limits to prevent overload:

- **Active per profile**: 1. A profile holds one `claimed` card at a time.
- **Review queue depth**: bounded (e.g., 5). When the human's `delivered` queue exceeds the limit, the Kanban dispatcher delays new card creation on that lane (or escalates a notification via Telegram, depending on configuration).
- **Synthesis lane**: bounded (e.g., 3). Too many synthesis cards at once means quality drops because evidence cannot be fully integrated.

These are operational tuning parameters, not architectural constants.

<!-- memoria-nav -->

---

[← Previous: Board, states, and the review gate](README.md)

[Next: Board schema and handoff (reference) →](card-schema.md)
