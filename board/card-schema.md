---
mode: reference
audience: operator
topic: board
---

# Board schema and handoff (reference)

Card fields, the entity vocabulary the board implicitly tracks, and the handoff pattern (prose summary + structured task packet). For the conceptual narrative see [README.md](README.md); for state values and the review gate see [states.md](states.md).

## Board schema fields

These are the recommended fields on every card. They are the API between the workers, the Verifier (when applicable), and the human.

| Field | Type | Purpose |
| --- | --- | --- |
| `status` | enum | Current board state (one of the nine above). |
| `assignee` | string | Profile currently holding the card. |
| `lane` | enum | Which lane the card belongs to. |
| `blocked_reason` | text | Why the card is blocked, when applicable. |
| `retry_count` | int | Number of recoverable failures so far. |
| `handoff_note` | text | Human-readable handoff summary (Blocker / Tried / Next). |
| `task_packet` | json | Structured [task packet](../glossary.md#board-and-cards) for the next worker. Self-contained — receiving worker needs nothing else to start. See [Handoff pattern](#handoff-pattern). |
| `last_updated` | timestamp | When state last changed. |
| `canonical_target` | path | Where the output should land if approved (e.g. `30-synthesis/01-claims/xyz.md`). |
| `review_status` | enum | Independent of `status`: `unreviewed`, `requested`, `in-review`, `approved`, `rejected`. |
| `review_owner` | string | Who owes the next review decision. |
| `review_requested_at` | timestamp | When the worker handed off for review. |
| `reviewed_at` | timestamp | When the human (or Verifier for the agent-recommendation phase) acted on the review. |

`review_status` is separate from `status` deliberately. A card can be `active` while `review_status` is still `unreviewed`. This lets dashboards and dispatch logic query review independently.

Both `status` and `review_status` are **board-card fields only** — notes never carry them. A note's lifecycle phase lives in `lifecycle` (with per-type refinements like `maturity`), whose value set is disjoint from `status`. See [frontmatter-schema.md](../vault/frontmatter-schema.md).

## Entity vocabulary

The board fields above implicitly track five distinct entities. Naming them is useful because queries, dashboards, and the audit log all touch the same vocabulary — and conflating them produces confusing reports ("how many handoffs happened this week?" is unanswerable if handoffs live inside a free-text field).

| Entity | What it is | Where it lives on the card |
| --- | --- | --- |
| **Task** | A unit of work. One card = one task. | The card itself; identified by `task_id`. |
| **Handoff** | An event where one profile passes work to another. Multiple handoffs per task are normal (Librarian creates source → human classifies → Writer drafts → Verifier traces). | The `task_packet` field carries the *current* handoff; comment log records the history. |
| **Artifact** | An output produced by the task — a paper note, an answer draft, a code module, a deliverable. | The `canonical_target` field points to the artifact path; multi-artifact tasks list paths in `task_packet.expected_outputs`. |
| **Verdict** | A review decision on the task: `approve`, `reject`, or `escalate` (see the verdict vocabulary below). Issued by the human (always for `status: approved`) or by Verifier for the agent-recommendation phase of a verify card. One per review pass. Routing after a `reject` verdict (discard vs supersede) is a separate human action, not part of the verdict itself. | The `review_status` field carries the latest verdict; comment log records prior verdicts. |
| **State transition** | A change in `status` (`ready → active`, `active → awaiting-review`, etc.). Multiple transitions per task. | Implicit in card history; `last_updated` records the most recent transition timestamp. |

### Why not a separate registry

The doc that surfaced this vocabulary (`memoria_task_registry_schema.md` in `raw/`) proposes a separate SQLite registry with one table per entity. Memoria does not adopt that — the Hermes built-in Kanban already holds task and transition state, and a parallel registry creates two sources of truth that have to be kept in sync.

Instead, the entities above are **projections of card state**, not separate stores:

- **Aggregate views** for the [fleet-observability dashboard](../dashboards/fleet-observability.md) (deferred to Post-MVS) come from a scheduled aggregator that reads card history and writes `lane-metric` and `skill-metric` notes to `00-meta/08-metrics/` — both the aggregator and the folder arrive when the dashboard is stood up; see [future-directions.md §"Fleet observability"](../roadmap/future-directions.md#fleet-observability).
- **Audit trail** for individual actions comes from `00-meta/02-logs/audit.jsonl` (written by the policy MCP).
- **Card detail** (task + current handoff + current verdict) lives on the card itself.

This keeps the board as the single source of truth while still giving the design clear vocabulary for what the card tracks.

## Handoff pattern

Handoffs carry two things: a **human-readable summary** for board readers, and a **structured task packet** for the next worker (and the policy MCP). Both live on the card.

### Human-readable summary

The `handoff_note` field is short prose with three lines. It is what the next worker or the human sees at a glance:

```text
Blocker: [what's stopping forward progress]
Tried: [what's been attempted; what was learned]
Next: [the exact action the next worker should take]
```

When the human spawns a revision card after a rejection, the new card's handoff note describes what was wrong with the original (the new card carries `supersedes: <original-id>` for provenance; the original closes with `outcome: superseded`). When a Librarian hands off to Verifier for filing-time similarity check, the same handoff format applies. When a Writer's draft fires the Verifier hook, the same.

### Structured task packet

The `task_packet` field is JSON. It is what the next worker (or a delegated child agent) consumes programmatically. Hermes delegated children return summaries rather than sharing live parent state, so every handoff packet must be **self-contained** — the receiving worker should need nothing beyond the packet to start work.

```json
{
  "task_id": "TASK-2026-05-25-014",
  "origin_profile": "human",
  "target_profile": "memoria-librarian",
  "goal": "Find recent systematic reviews on persuasive digital health interventions",
  "context": {
    "research_direction": "digital health coaching",
    "project": "memoria-health-coaching"
  },
  "allowed_paths": [
    "10-inbox/03-candidates/**",
    "20-sources/01-papers/**"
  ],
  "expected_outputs": [
    "candidate paper notes",
    "classification proposal",
    "handoff note"
  ],
  "review_checks": [
    "stable identifier present",
    "proposed classification included"
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
- `review_checks` — what the agent (Verifier, Linter) or the human will verify before approving.
- `next_state` — the exit state the worker should move the card to when complete.

The `handoff_note` and `task_packet` together form the durable trail. The conversation does not.

### Why both forms

The prose `handoff_note` is for humans scanning the board; the structured `task_packet` is for the next worker (and tools that read the card). Trying to use one for both produces packets too verbose to scan and prose too vague to act on programmatically. Keep them separate.

<!-- memoria-nav -->

---

[← Previous: Board states and review gate (reference)](states.md)

[Next: Hermes profiles →](../profiles/README.md)
