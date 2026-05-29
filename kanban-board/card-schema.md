---
mode: reference
audience: operator
topic: board
---

# Board schema and handoff (reference)

The board **is** the Hermes built-in Kanban. Its task schema is fixed and not user-customizable, so Memoria does not define its own card columns — it uses Hermes' built-in fields and stores everything Memoria-specific inside the free-form `metadata` JSON. This page lists the real Hermes fields Memoria relies on, the `metadata` conventions Memoria layers on top, the entity vocabulary the board tracks, and the handoff pattern. For the conceptual narrative see [README.md](README.md); for the state machine and the review gate see [states.md](states.md).

The authoritative schema is the upstream Hermes Kanban reference ([tools-reference#kanban-toolset](https://hermes-agent.nousresearch.com/docs/reference/tools-reference#kanban-toolset), [cli-commands#hermes-kanban](https://hermes-agent.nousresearch.com/docs/reference/cli-commands#hermes-kanban)). If a field below conflicts with the live docs, the live docs win.

## Hermes built-in card fields

These are real, fixed columns on every Hermes Kanban task. Memoria reads and writes them through the kanban toolset and CLI; it cannot rename or add to them.

| Field | Type | Purpose | Memoria use |
| --- | --- | --- | --- |
| `task_id` | string | Unique identifier (e.g. `t_abcd`). | The card identity; referenced from `metadata` and handoff payloads. |
| `title` | string (required) | Short task name. | One line describing the card's goal. |
| `body` | markdown | Longer description. | The specification produced during `triage` (`hermes kanban specify`). |
| `assignee` | string | Profile name handling the work; `none` when unassigned. | **Also the lane key** — the dispatcher routes by `assignee`. There is no separate `lane` field. |
| `status` | enum | Execution state (fixed enum below). | The execution lifecycle. Review state is tracked separately in `metadata` — see [states.md](states.md). |
| `priority` | int | Numeric priority. | Lane dispatch ordering. |
| `tenant` | string | Optional namespace for multi-tenant isolation. | Project / research-direction scoping. |
| `created_at` | timestamp | When the card was created. | Provenance. |
| `scheduled_at` | timestamp | Optional future dispatch time. | Deferred work (`hermes kanban schedule`). |
| `workspace` | enum | `scratch`, `dir:<path>`, or `worktree:<path>`. | Where the worker operates. |
| `branch` | string | Optional git branch. | Used by the Coder lane. |
| `max_runtime_seconds` | int | Per-run wall-clock ceiling. | Lane timeout. |
| `max_retries` | int | Circuit-breaker override for recoverable failures. | Replaces the old `retry_count` notion — Hermes counts retries in run history, not a card field. |
| `idempotency_key` | string | Deduplication key for retried automation. | Prevents duplicate cards from upstream triggers. |
| `parents` | array | Parent task IDs (dependency edges via `kanban_link`). | Inter-card dependencies. |

### Status enum (fixed)

```text
triage · todo · ready · running · blocked · done · archived
```

These seven values are the only legal `status` values. Memoria's conceptual states (`queued`, `delivered`, `accepted`, …) are **not** stored here — they map onto this enum plus the review overlay. See the [state crosswalk in states.md](states.md#crosswalk-old-memoria-states).

## Run-level handoff fields

When a worker finishes or blocks, the payload lives on the **run**, not as task columns. These come from the kanban toolset (`kanban_complete`, `kanban_block`).

| Field | Where | Purpose |
| --- | --- | --- |
| `summary` | `kanban_complete` | Human-readable completion note. Memoria writes the Blocker / Tried / Next prose here. |
| `metadata` | `kanban_complete` | Free-form JSON: structured evidence (`changed_files`, `decisions`, `tests_run`, …) **and** Memoria's overlay fields below. |
| `reason` | `kanban_block` | Why the task is blocked, for human input. Replaces the old `blocked_reason`. |
| `outcome` | run | Hermes run-execution result — a closed, Hermes-defined enum per the [live Kanban docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban): `completed`, `blocked`, `crashed`, `gave_up`, `reclaimed`, `timed_out`, `spawn_failed`, `protocol_violation`. Records what happened on a *run*. **Archival reasons are not outcomes** — see `metadata.archive_reason` below. |
| `error` | run | Failure detail when applicable. |
| `worker_context` | run | Prior attempts (previous runs' outcome / summary / error / metadata) plus parent results. |

## Memoria overlay fields (inside `metadata`)

Because the Hermes schema is fixed, Memoria's review gate and provenance live as **conventions inside the `metadata` JSON** — not as card columns. Workers and the policy MCP read and write these keys; dashboards query them via Dataview over exported card state.

| `metadata` key | Type | Purpose |
| --- | --- | --- |
| `review_status` | enum | The human review lifecycle, independent of `status`: `unreviewed`, `requested`, `in-review`, `approved`, `rejected`. `approved` is the canonical human acceptance (the old `status: accepted`). |
| `agent_verdict` | enum | Optional agent recommendation (Verifier, Linter): `approve`, `reject`, `escalate`, plus the Verifier triple `verify-clean` / `verify-needs-revision` / `verify-needs-attention`. Feeds, but does not replace, the human `review_status`. |
| `review_owner` | string | Who owes the next review decision. |
| `review_requested_at` | timestamp | When the worker handed off for review. |
| `reviewed_at` | timestamp | When the human (or an agent, for its recommendation) acted on the review. |
| `promote_target` | path | Where the output should land **if approved** (e.g. `30-synthesis/01-claims/xyz.md`). |
| `supersedes` | task_id | On a revision card, the original card it replaces (the original is archived with `metadata.archive_reason: superseded`). |
| `archive_reason` | enum | On an archived card, why it was archived: `superseded` (replaced by a successor card) or `discarded` (rejected with no successor). Hermes archiving is a status transition with no native reason field, so Memoria records the reason here rather than in `outcome`. |

`review_status` is deliberately separate from `status`. A card can be `status: running` while `review_status` is still `unreviewed`. This lets dashboards and dispatch logic query review independently. Both are **board-card concerns only** — notes never carry them. A note's lifecycle phase lives in `lifecycle` (with per-type refinements like `maturity`), whose value set is disjoint from the board's. See [frontmatter-schema.md](../vault/frontmatter-schema.md).

## Crosswalk: old field names → Hermes reality

Earlier drafts of this doc described a custom card schema. Those field names map onto Hermes as follows; other docs that still use the old names should be read through this table until they are updated.

| Old Memoria field | Hermes reality |
| --- | --- |
| `status` | Unchanged — Hermes built-in, but uses the fixed enum above (not Memoria's nine states). |
| `assignee` | Unchanged — Hermes built-in; **also the lane key**. |
| `lane` | Dropped — derived from `assignee`; not a field. |
| `blocked_reason` | `reason` (the `kanban_block` argument). |
| `retry_count` | `max_retries` + run history; not a stored count. |
| `handoff_note` | `summary` (the `kanban_complete` prose note). |
| `task_packet` | `metadata` (the `kanban_complete` JSON payload). |
| `last_updated` | No field — derived from the card event stream (`hermes kanban tail`). |
| `promote_target` | `metadata.promote_target`. |
| `review_status`, `review_owner`, `review_requested_at`, `reviewed_at` | `metadata.*` keys (see overlay table). |

## Entity vocabulary

The fields above implicitly track five distinct entities. Naming them is useful because queries, dashboards, and the audit log all touch the same vocabulary — and conflating them produces confusing reports ("how many handoffs happened this week?" is unanswerable if handoffs live inside a free-text field). For overloaded *words* rather than entities — `review`, `verdict`, `promote`, `lane`, `canonical` — see the [glossary](../glossary.md).

| Entity | What it is | Where it lives on the card |
| --- | --- | --- |
| **Task** | A unit of work. One card = one task. | The card itself; identified by `task_id`. |
| **Handoff** | An event where one profile passes work to another. Multiple handoffs per task are normal (Librarian creates source → human classifies → Writer drafts → Verifier traces). | A `kanban_complete` / `kanban_create` event carrying `summary` + `metadata`; comment log records the history. |
| **Artifact** | An output produced by the task — a paper note, an answer draft, a code module, a deliverable. | `metadata.promote_target` points to the artifact path; multi-artifact tasks list paths in `metadata.expected_outputs`. |
| **Verdict** | A review decision on the task: `approve`, `reject`, or `escalate` (see the [verdict vocabulary](states.md#review-verdict-vocabulary)). Issued by the human (always for canonical acceptance) or recommended by Verifier/Linter. One per review pass. Routing after a `reject` (discard vs supersede) is a separate human action, not part of the verdict. | `metadata.review_status` carries the human decision; `metadata.agent_verdict` carries the agent recommendation; comment log records prior verdicts. |
| **State transition** | A change in `status` (`ready → running`, `running → done`, etc.). Multiple transitions per task. | The card event stream; `hermes kanban tail` follows it. |

### Why not a separate registry

The doc that surfaced this vocabulary (`memoria_task_registry_schema.md` in `raw/`) proposes a separate SQLite registry with one table per entity. Memoria does not adopt that — the Hermes built-in Kanban already holds task and transition state (in its own `kanban.db`), and a parallel registry creates two sources of truth that have to be kept in sync.

Instead, the entities above are **projections of card state**, not separate stores:

- **Aggregate views** for the [fleet-health dashboard](../dashboards/fleet-health.md) (deferred to Post-MVS) come from a scheduled aggregator that reads card history and writes `lane-metric` and `skill-metric` notes to `00-meta/08-metrics/` — both the aggregator and the folder arrive when the dashboard is stood up; see [future-directions.md §"Fleet observability"](../roadmap/future-directions.md#fleet-observability).
- **Audit trail** for individual actions comes from `00-meta/02-logs/audit.jsonl` (written by the policy MCP).
- **Card detail** (task + current handoff + current verdict) lives on the card itself, in Hermes' `kanban.db`.

This keeps the board as the single source of truth while still giving the design clear vocabulary for what the card tracks.

## Handoff pattern

Handoffs carry two things: a **human-readable summary** for board readers, and a **structured payload** for the next worker (and the policy MCP). In Hermes these are the `summary` and `metadata` arguments of `kanban_complete` — both travel with the run.

### Human-readable summary

The `summary` field is short prose with three lines. It is what the next worker or the human sees at a glance:

```text
Blocker: [what's stopping forward progress]
Tried: [what's been attempted; what was learned]
Next: [the exact action the next worker should take]
```

When the human spawns a revision card after a rejection, the new card's `summary` describes what was wrong with the original (the new card carries `metadata.supersedes: <original-id>` for provenance; the original is archived with `metadata.archive_reason: superseded`). When a Librarian hands off to Verifier for a filing-time similarity check, the same summary format applies. When a Writer's draft fires the Verifier hook, the same.

### Structured payload

The `metadata` field is JSON. It is what the next worker (or a delegated child agent) consumes programmatically. Hermes delegated children return summaries rather than sharing live parent state, so every handoff payload must be **self-contained** — the receiving worker should need nothing beyond the payload to start work.

```json
{
  "task_id": "t_2b14",
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
    "handoff summary"
  ],
  "review_checks": [
    "stable identifier present",
    "proposed classification included"
  ]
}
```

Field meanings:

- `task_id` — links back to the card. Required.
- `origin_profile` / `target_profile` — who handed off, who receives. Required.
- `goal` — one sentence describing the outcome the receiving worker is responsible for.
- `context` — structured key/value pairs the worker needs (project, research_direction, related cards). Not free prose.
- `allowed_paths` — the worker's write scope for this card specifically. The policy MCP cross-checks against the lane override; the payload can narrow but never widen.
- `expected_outputs` — what the receiving worker must produce before the card can exit. Verifier or Linter checks these where applicable.
- `review_checks` — what the agent (Verifier, Linter) or the human will verify before approving.

The exit `status` is set by the lifecycle terminator the worker calls (`kanban_complete` → `done`, `kanban_block` → `blocked`), not by a field in the payload.

The `summary` and `metadata` together form the durable trail. The conversation does not.

### Why both forms

The prose `summary` is for humans scanning the board; the structured `metadata` is for the next worker (and tools that read the card). Trying to use one for both produces payloads too verbose to scan and prose too vague to act on programmatically. Keep them separate.
