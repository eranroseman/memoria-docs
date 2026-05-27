# Memory tiers

Hermes profiles each have their own memory, but "memory" is not one thing — it operates at four distinct scopes. Confusing the scopes is the source of most "the agent forgot" and "the agent remembered something it shouldn't have" bugs. The four tiers:

| Tier | Scope | Lifespan | Who writes | What it holds |
| --- | --- | --- | --- | --- |
| **Profile memory** | One Hermes profile | Session-bound; cleared on `/clear` | The profile itself | Short-term working state — current goal, recent tool results, in-flight reasoning. |
| **Lane memory** | One lane (across profiles in that lane) | Card-bound; lives with the active card | The lane's primary profile | Current lane context — the task packet, recent handoff notes, the working set of source notes. |
| **Project memory** | One project, across lanes | Project-bound; lives in `40-workbench/01-projects/<project>/` | All profiles that touch the project | The project's research-directions, ongoing questions, decisions log. Shared, durable, but project-scoped. |
| **Episodic archive** | The vault | Indefinite; never overwritten, only appended | The linter (sole writer) | Snapshots, weekly summaries, fleet metrics, archived audit logs. Long-term operational record. |

## Rules

- **Profile memory is not shared.** The researcher's session memory does not bleed into the writer's session. Each profile starts a session with only its lane memory + project memory loaded.
- **Lane memory is per-card, not per-profile.** When a card moves from research to writer, the lane memory travels via the task packet (see [03-board.md](../03-board.md#structured-task-packet)). The writer doesn't inherit the researcher's session scratchpad.
- **Project memory is the cross-lane channel.** Anything that needs to survive across lanes belongs here — not in profile memory (too local) and not in the episodic archive (too distant). The `research-directions.md` file at the project root is the canonical example.
- **The episodic archive is append-only.** Profiles read it; only the linter writes to it. This is what `00-meta/04-logs/audit.jsonl` and `00-meta/08-metrics/` are part of.

## Why the tiering matters

Without these distinctions, every cross-session question collapses into "store it in memory and hope" — which means profiles either share too much (leaking context between lanes) or too little (re-deriving the project goal every session). The tiers make the right answer explicit per question: "Where does X live?" has a single right answer based on its lifespan and scope.

## Related

- Task packet (how lane memory travels between profiles): [03-board.md](../03-board.md#structured-task-packet)
- Project research-directions pattern: [04-workflows.md](../04-workflows.md)
- Audit log location: `00-meta/04-logs/audit.jsonl` (see [reference/policy-mcp.md](../reference/policy-mcp.md))
