---
mode: explanation
audience: operator
topic: profiles
---

# Hermes profiles

Hermes is the worker layer, split into seven specialist profiles. Each has a focused mission, narrow folder permissions, a small command surface, the skills it actually needs, and an explicit set of MCPs / external tools. **Profiles never share permissions by default — every privilege is granted, not inherited.** This document is the authoritative reference for what each profile can do, what it cannot do, and how the seven relate to each other.

## What's in this document

**What the profiles are** — [The seven profiles](#the-seven-profiles), [Core design rule](#core-design-rule) (the load-bearing principle).

**Permissions and constraints** — [Lane permissions matrix](#lane-permissions-matrix), [Autonomy levels](#autonomy-levels), [Folder permission matrix](#folder-permission-matrix), [Lane-override files](#lane-override-files) (the YAML encoding).

**Dispositions and dynamics** — [Delegation ladder](#delegation-ladder), [Routing without an Orchestrator](#routing-without-an-orchestrator), [Inter-profile handoff patterns](#inter-profile-handoff-patterns).

**Extension and per-profile detail** — [Skill governance (deferred)](#skill-governance-deferred), [Per-profile contracts](#per-profile-contracts), [Coder ↔ external coding agent](#coder--external-coding-agent), [Commands](#commands).

**Operational concerns** — [Workflow ownership](#workflow-ownership), [Scaling](#scaling), [Anti-patterns](#anti-patterns).

## The seven profiles

| Profile | Primary lane | Autonomy level | One-line mission |
| --- | --- | --- | --- |
| **Librarian** | Library | Level 1–2 (background + Kanban) | Find, ingest, enrich, and classify evidence. Optimistic. |
| **Mapper** | Mapping | Level 2 (Kanban-pulled) | Map the corpus for a project: scope reports, gap reports, cluster density. Read-only across vault. |
| **Socratic** | Socratic | Level 3 (interactive) | Sharpen the human's thinking through questioning and lens-based reading. Architecturally write-denied. |
| **Writer** | Writer | Level 2 with review gate | Turn evidence into drafts, answer notes, and reference-ready prose. |
| **Verifier** | Verify | Level 2 (Kanban-pulled) | Trace claims to sources; verify citations; flag duplicates and retractions. Read-only across vault except verification reports. |
| **Coder** | Coder | Level 2 (external dispatch) | Build and maintain code artifacts, scripts, repos. Transactional. |
| **Linter** | Linter | Level 1 (scheduled) | Validate structure, metadata, schema, link health. Default dry-run. Also owns session-log and audit-trail housekeeping. |

**No Orchestrator, no Reviewer.** Routing lives in lane-overrides and Kanban dispatch — see [Routing without an Orchestrator](#routing-without-an-orchestrator). The review gate rides on the card's `review_status` (a `done` card with `review_status: requested`, promoted to `approved`) enforced by the policy MCP; the mechanical parts of review (claim tracing, similarity, retraction) live in Verifier, the judgment parts with the human.

## Lane permissions matrix

The **consolidated lane view** — what each lane may run (skills), what tools it uses, what's denied, and where it can write. It's the human-readable summary you check first when a worker behaves unexpectedly. A complementary [folder × profile view](#folder-permission-matrix) appears below; the YAML form the policy MCP actually reads is in [lane-override files](#lane-override-files).

| Lane | Primary role | Core commands | Allowed skills | Allowed tools | Denied capabilities | Write scope |
| --- | --- | --- | --- | --- | --- | --- |
| **Library** | Find and ingest evidence | `find`, `ingest`, `enrich`, `classify`, `query` | `paper-lookup`, `arxiv-search`, `pyzotero`, `citation-management`, `literature-review`, `obsidian-paper-note`, `rest-passthrough` | `search_web`, `fetch_url`, vault read/write | canonical publish, destructive shell, unrestricted HTTP | `10-inbox/`, `20-sources/` |
| **Mapping** | Map the corpus | `scope-project`, `gap-report`, `cluster-map`, `comparative-brief` | `scope-project`, `gap-report`, `cluster-mapping`, `comparative-brief` | vault read | all write tools, external APIs, drafting | `40-workbench/01-projects/*/map/corpus-map.md`, `*/gap-report.md`, `*/comparative-briefs/*`, `*/cluster-maps/*` |
| **Socratic** | Question without producing | `socratic-processing`, `lens-reading` | `socratic-processing`, `lens-reading` (parameterized: `mamykina-lens`, `veinot-equity-lens`, etc.) | vault read | **all write tools** (`policy.allow.write: []`), external APIs, drafting, queue dispatch (`routing.invocation: interactive_only`) | (none — `read_only_mode`, hard) |
| **Writer** | Draft and synthesize | `draft`, `query`, `lint`, `promote` (handoff) | `llm-wiki draft`, `note-refactor`, `scientific-writing`, `counter-outline` | `search_web`, `fetch_url`, vault read/write | `rest-passthrough`, external-API skills, publish-canonical | `10-inbox/02-answers/`, `40-workbench/01-projects/*/drafts/`, `40-workbench/01-projects/*/framing/` (via `counter-outline`) |
| **Verify** | Verify claims, citations, duplicates | `cite-check`, `similarity-check`, `find-duplicates`, `retraction-check` | `cite-check`, `similarity-check`, `find-duplicates`, `retraction-check` | `search_web`, `fetch_url`, vault read | all write tools except verification reports and gap-candidate cards, drafting | `40-workbench/01-projects/*/verification/*`, `10-inbox/03-candidates/` (gap-candidate cards only) |
| **Linter** | Validate, report, and log | `lint`, `schema-check`, `schema-migrate`, `health-report`, `graph-analyze`, `session-log`, `dry-run`, `report` | `schema-check`, `graph-analyze`, `health-report`, `session-log` | vault read, file indexer, Git | canonical edits, schema-content auto-fixes, work spawning | `00-meta/02-logs/` (audit and session logs only), dry-run reports |
| **Coder** | Code artifacts | `code`, `commit`, `revert`, `workspace`, `scaffold` | `scaffold-code-note`, `workspace-coordinate`, `commit-and-document` (thin Hermes-side wrappers; substantive coding work lives in the external coding agent — see [Coder ↔ external coding agent](#coder--external-coding-agent)) | Git, filesystem, repo APIs | canonical edits, prose ownership | `40-workbench/01-projects/*/code/` |

Rules of thumb:

- **Networked skills are lane-restricted.** Only Library can call `rest-passthrough`. The passthrough is the escape hatch (see [architecture/capability-stack.md](../architecture/capability-stack.md#rest-passthrough--the-escape-hatch)); confining it to one lane keeps external I/O auditable.
- **Socratic and Mapper are read-only.** Any `decision: allow` for `profile: memoria-socratic` or `profile: memoria-mapper` on a `write` action outside its declared scratch path is a configuration bug — surfaced by the [audit-log dashboard](../dashboards/audit-log.md).
- **No lane writes to canonical zones.** `30-synthesis/01-claims/`, `30-synthesis/02-reference/`, `30-synthesis/03-moc/`, and `50-deliverables/` are policy-MCP `dry_run` for every lane. Promotion is always synchronous with human attention.

## Autonomy levels

Each profile's autonomy level is part of its contract:

- **Level 1 (background)** — runs unattended on a cron schedule. Produces reports; never acts on canonical content. Linter is the definitive example.
- **Level 2 (Kanban-pulled)** — picks up cards from its lane queue. Produces output to review-gated paths. The bulk of Memoria's work.
- **Level 2 with review gate** — produces drafts that don't promote to canonical without explicit human approval. Writer.
- **Level 3 (interactive)** — invoked synchronously by the human (via ACP, command palette, or Telegram). No queue. Socratic.
- **Level 2 (external dispatch)** — handoffs to an external agent (Claude Code, Aider, Codex) via handoff payloads. Coder.

The level governs both the cadence (background / kanban-pulled / interactive) and the expected output mode (report / artifact / conversation). It does not loosen any policy MCP rule.

## Folder permission matrix

This is the operational access map. For the layered folder structure these columns refer to, see [vault/README.md](../vault/README.md).

| Profile | 00-meta | 10-inbox | 20-sources | 30-synthesis/01-claims | 30-synthesis/02-reference | 30-synthesis/03-moc | 40-workbench | 50-deliverables |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Librarian | Read | Write (discovery, candidates) | Write (create, enrich) | Read only | Read for context | Read for context | Read | Read only |
| Mapper | Read | Read only | Read only | Read only | Read only | Read only | Write (`01-projects/*/corpus-map.md`, `*/gap-report.md`, `*/comparative-briefs/*`, `*/cluster-maps/*`) | Read only |
| Socratic | Read | Read only | Read only | Read only | Read only | Read only | Read only | Read only |
| Writer | Read | Write (answer drafts) | Read only | Read only | Write drafts (review-gated) | Read; suggest | Write (drafts, framing) | Read only unless export task |
| Verifier | Read | Write (gap-candidate cards in `03-candidates/`) | Read only | Read only | Read only | Read only | Write (`01-projects/*/verification/*`) | Read only |
| Coder | Read | Read only | Read only | Read only | Read for context | Read | Write (`40-workbench/01-projects/*/code/`, project pages under `01-projects/`) | Read / write on explicit export tasks |
| Linter | Read; write `02-logs/` (audit, session) | Read | Read | Read | Read | Read | Read | Read |

Rule of thumb: **canonical synthesis remains human-owned** across the `30-synthesis/` canonical zones (`01-claims/`, `02-reference/`, `03-moc/`). **Schema governance remains human-owned** in `00-meta/` except for the logs subfolder Linter writes to. Project scratch (`40-workbench/01-projects/`) is the only zone where multiple profiles write, and each profile writes to its own named subfolder.

## Lane-override files

The folder permission matrix above is the human-readable summary. The machine-readable encoding lives in **lane-override files** at `.memoria/lane-overrides/{lane}.yaml`. One file per lane. The policy MCP reads these at startup and enforces them at every write (see [architecture/policy-mcp.md](../architecture/policy-mcp.md)).

### File shape

Each lane-override has the same four blocks. Example for the library lane:

```yaml
lane: library
profile: memoria-librarian

policy:
  allow:
    skills:
      - paper-lookup
      - pyzotero
      - citation-management
    write:
      - "10-inbox/**"
      - "20-sources/**"
  deny:
    skills:
      - canonical_publish
    write:
      - "30-synthesis/**"
      - "50-deliverables/**"
  require:
    - audit_log
    - timeout_required
    - source_tracking

routing:
  external_api_policy: explicit_only        # blocked | explicit_only | open
  write_scope:                               # the lane's authoritative short-list
    - "10-inbox/"
    - "20-sources/"
```

The four blocks:

- **`policy.allow`** — skills the lane may load and paths it may write. Patterns are glob-style relative to the vault root.
- **`policy.deny`** — explicit blocks. Deny wins over allow.
- **`policy.require`** — invariants the worker must honor. Common values: `audit_log` (every action logged), `timeout_required` (no unbounded calls), `source_tracking` (writes carry a provenance trail), `read_only_mode` (no writes at all — used by Socratic and Mapper), `canonical_write_gate` (writes to canonical zones always degrade to `dry_run`).
- **`routing.write_scope`** — the authoritative short-list, usually narrower than `policy.allow.write`. Used by the dispatch rules to decide where a worker's output should land by default.
- **`routing.invocation`** — how the lane is invoked. `dispatched` (default; the Kanban dispatcher pulls cards from this lane's queue) or `interactive_only` (the lane is never queue-dispatched; only synchronous human invocation reaches it). Socratic uses `interactive_only` so a misconfigured cron job can't queue work onto a write-denied conversational profile.

### One file per lane

| Lane | File | Notes |
| --- | --- | --- |
| Library | `.memoria/lane-overrides/library.yaml` | May call external APIs through approved skills. |
| Mapping | `.memoria/lane-overrides/mapper.yaml` | `read_only_mode` for the vault; project-scratch writes only. |
| Socratic | `.memoria/lane-overrides/socratic.yaml` | Hard `policy.allow.write: []` — write-denied across the vault. `routing.invocation: interactive_only` — never queue-dispatched. |
| Writer | `.memoria/lane-overrides/writer.yaml` | May draft and refine; no direct external API; canonical writes degrade to `dry_run`. |
| Verify | `.memoria/lane-overrides/verifier.yaml` | `read_only_mode` for the vault; verification-report writes only. |
| Coder | `.memoria/lane-overrides/coder.yaml` | Writes to `40-workbench/01-projects/*/code/` only. |
| Linter | `.memoria/lane-overrides/linter.yaml` | Dry-run by default; `auto_fix` is class-gated by the policy MCP; owns `00-meta/02-logs/` for session and audit-trail housekeeping. |

### Why not encode this in `config.yaml`

The profile-level `config.yaml` says what the profile *is* — model, command surface, skill registry, base MCPs. The lane-override says what the profile *may do in this vault*. The split lets the same profile run against multiple vaults with different policies, and it lets us review permissions independently of model and tooling.

### Composability

Two files compose at session start:

1. `~/.hermes/profiles/memoria-<profile>/config.yaml` — model, command surface, skill registry, base MCPs.
2. `.memoria/lane-overrides/<lane>.yaml` — policy and routing for this vault.

If a worker would claim a card outside its lane override, the policy MCP returns `deny`. If a worker tries to write into a canonical zone, the MCP returns `dry_run` and the action becomes a board comment for the human to act on. Either way, nothing canonical changes without an explicit approval step.

## Skill governance (deferred)

**Status: deferred — maybe later if needed.** Adding a skill today is purely a runtime operation: edit the relevant lane-override file's `policy.allow.skills` list and drop the `SKILL.md` into the right `.memoria/profiles/memoria-<name>/skills/` folder. The richer lifecycle discipline — state machine, per-skill governance notes in `00-meta/07-skills/`, the `skill-lifecycle` dashboard, the 7-step onboarding checklist, and the passthrough-to-dedicated-skill graduation thresholds — is a future possibility documented in [roadmap/skill-governance.md](../roadmap/skill-governance.md) but not active in the current design.

### Skills with restrictive policy

Most skills inherit their host lane's policy unchanged. A small set of skills *tighten* it further by declaring additive `policy.deny` rules in their SKILL.md frontmatter. The mechanism (composition semantics, enforcement path, frontmatter contract) lives in [architecture/policy-mcp.md](../architecture/policy-mcp.md#skill-conditional-policy); the **catalog** of Memoria's restrictive skills lives here:

| Skill | Loaded by | What it adds | Why |
| --- | --- | --- | --- |
| `counter-outline` | Writer | `deny: write:30-synthesis/**`, `deny: write:50-deliverables/**`, `deny: write:10-inbox/**` — scratch-only | Generates 2–3 competing project outlines before drafting begins. Outputs land in `40-workbench/01-projects/*/framing/` only; "first outline wins by default" is the failure mode this prevents. Loaded only when Writer is in the Frame stage of a downstream pipeline. |

The shared property: a skill's deny rules are stricter than the host lane's. A `counter-outline` session under Writer is more restricted than Writer's baseline, never more permissive — and the narrowing is enforced at the policy MCP, not by prompt discipline.

**Note on what's NOT here.** Earlier versions of this design had `socratic-processing` and `lens-reading` as restrictive skills. They've been promoted to a profile (Socratic) whose lane policy is `policy.allow.write: []` natively. Profile-level enforcement is stricter than skill-level enforcement because it can't be sidestepped by loading the host profile without the skill. The skill-conditional-policy mechanism is preserved for `counter-outline` and any future scratch-only skills that are too narrow to justify their own profile.

## Per-profile contracts

Each profile's operational contract — exact allowed folders, commands, skills, tooling, rules, and exit conditions — lives in its own SOUL.md file in the starter vault. These are the runtime contracts that Hermes reads at session start; the design documents summarize, the SOUL.md files specify.

| Profile | Design summary | Runtime contract (SOUL.md, in starter vault) |
| --- | --- | --- |
| Librarian | [profiles/librarian.md](librarian.md) | `.memoria/profiles/memoria-librarian/SOUL.md` |
| Mapper | [profiles/mapper.md](mapper.md) | `.memoria/profiles/memoria-mapper/SOUL.md` |
| Socratic | [profiles/socratic.md](socratic.md) | `.memoria/profiles/memoria-socratic/SOUL.md` |
| Writer | [profiles/writer.md](writer.md) | `.memoria/profiles/memoria-writer/SOUL.md` |
| Verifier | [profiles/verifier.md](verifier.md) | `.memoria/profiles/memoria-verifier/SOUL.md` |
| Coder | [profiles/coder.md](coder.md) | `.memoria/profiles/memoria-coder/SOUL.md` |
| Linter | [profiles/linter.md](linter.md) | `.memoria/profiles/memoria-linter/SOUL.md` (plus `M-detectors.md`) |

The tables in this document — folder permission matrix, command summary, lane mapping, delegation ladder, workflow ownership — are the architectural *summary*. For the specific permission lists, dry-run defaults, and exit conditions per profile, read the SOUL.md.

## Coder ↔ external coding agent

The Coder profile is **narrow by design**: it scaffolds `code-note` handoffs and documents provenance. The actual editing happens in a specialized external coding agent (Kilocode, Aider, Claude Code, Codex) running as a peer with a shared filesystem — the vault is its read-only context, `40-workbench/01-projects/*/code/` is its write zone, and the human reviews `code-note` updates as the review gate. No orchestration infrastructure is needed: the two agents coordinate through the markdown handoff, not through subprocess dispatch.

**Full mechanism** — the two-agent boundary, VS Code workspace JSON for "small scripts in the vault" vs "real projects in their own repo," the `files.readonlyInclude` boundary, agent-instruction-file placement (`CLAUDE.md` / `.kilocode/rules/` / `.cursorrules`), the rule of thumb for when a project earns its own repo, and the `repo:` frontmatter convention on the `code-note` — lives in [profiles/why-coder-external-agent.md](why-coder-external-agent.md). The same pattern generalizes to **external rendering agents** (open-design) for visual deliverables, with `50-deliverables/` as the write zone and the `deliverable` note as the handoff artifact.

## Commands

Each profile's core verbs are listed in the [Lane permissions matrix](#lane-permissions-matrix) above. The full operational command catalog — every command, owner profile, and dry-run default — lives in [profile-commands.md](profile-commands.md). The rule: any command that writes to a canonical folder or runs a migration must default to dry-run.

## Routing without an Orchestrator

Routing — "which profile picks up this card?" — is rule-encoded, not reasoned. Three mechanisms together:

1. **The card's `assignee` (its lane)** determines which profile is eligible to claim it. A card assigned to the mapping lane can only be claimed by a worker whose lane-override file declares it on that lane. There is no separate `lane` field — the lane *is* the `assignee`.
2. **The lane-override file's `routing.write_scope`** declares where that worker's output should land by default, so the worker doesn't have to decide.
3. **The Kanban dispatcher** picks the highest-priority eligible card from each lane's queue when a worker becomes available. Priority is set at card creation (cron-triggered work is usually low-priority; human-initiated is high).

There is no reasoning step. If the rules can't decide (two profiles eligible, or none eligible), the card sits in `ready` (unclaimed) until the human intervenes. This is the design — silent reasoning about routing is exactly what the policy MCP exists to prevent.

When a card needs to move between lanes (e.g., from library to verify after ingest), the *worker that just finished* moves it. The Librarian who completes an ingest moves the card to the `verify` lane's queue if a similarity check is needed; the Writer who commits a draft fires the verify-on-commit hook. No central agent makes these decisions; the workflow does.

## Delegation ladder

Delegation is strongest in profiles that produce derivative artifacts (drafts, code-notes), weakest in profiles whose value depends on independence or non-production.

| Profile | Delegation posture |
| --- | --- |
| Librarian | **Targeted.** Delegates narrow enrichment or source lookups; keeps discovery ownership. |
| Mapper | **Low.** Delegates only mechanical retrieval (e.g., to qmd for vector search). Keeps the map authorship. |
| Socratic | **None.** Socratic has nothing to delegate — questions and conversations are the whole product. Cannot delegate writing because it can't write. |
| Writer | **Supportive.** Delegates facts or cleanup; keeps synthesis ownership. Final draft remains local. |
| Verifier | **Very low.** Delegation weakens verification independence. Inspects, traces, flags. |
| Coder | **Moderate.** Delegates helper work, formatting, lookup; keeps implementation control. Commits stay per task. Also delegates substantive coding work to the external coding agent via handoff payloads — see the Coder section above. |
| Linter | **Lowest.** Does not spawn work. May request context, but its job is to validate, report, and log. |

Rule: delegate narrow, temporary, low-risk subtasks; never delegate away the role's defining judgment.

## Workflow ownership

Each workflow has a primary profile. The authoritative view — workflows × profiles, with primary / support / no-claim distinctions across all 18 workflows — is the [Role × stage matrix in workflows/README.md](../workflows/README.md#role--stage-matrix). Don't duplicate it here; keep one source of truth.

## Inter-profile handoff patterns

The recommended handoff chain for a typical upstream card (source → claim):

```text
Librarian  ──► [classify-pending]  ──► (human classifies)  ──► [discuss-pending]
                                                                  │
                                       (human switches to Socratic; conversation; no writes)
                                                                  │
                                       (human writes claim note in Writer profile, with Verifier doing similarity-check at filing time)
                                                                  │
                                                                  └──► [distilled]
```

The recommended handoff chain for a typical downstream card (project → deliverable):

```text
Mapper  ──► [scope-review]  ──► (human decides project is ready)  ──► Writer with counter-outline
                                                                                          │
                                                                                          └──► [framing-review] ──► (human chooses framing)
                                                                                                                          │
                                                                                                                          └──► Writer (draft)
                                                                                                                                  │
                                                                                                                                  └──► Verifier (on commit) ──► [revise or export]
```

Linter can act at any point, on any card, with a dry-run report attached as a comment. Socratic is invoked synchronously by the human and never appears in queue-based handoff chains. The Coder operates in parallel for code artifacts.

## Core design rule

**Profiles own outcomes; lanes own claimability.**

A profile is a durable identity with a domain — librarians discover, mappers chart, writers synthesize, verifiers verify. The profile is responsible for the quality of its outputs.

A lane is a board-level contract about *who can claim a card*. The lane tells the dispatcher which profile class is allowed to move a card forward; the exit state is part of the lane contract.

The two work together:

- If a card is in the library lane, only Librarian-class workers may claim it.
- If a card is `done` and awaiting review (`review_status: requested`), only the human may clear it (no Reviewer profile to do so).
- When a worker finishes its slice, it completes the card to `done` with `review_status: requested` — it does not mark the work approved.

This is what prevents "everything becomes orchestration." Each profile stays accountable for what it ships.

## Scaling

The seven-profile design is the full Memoria design, but the system supports graduated starts: a **mode-based** single Hermes for the simplest setup, a **four-profile** minimal configuration (Librarian + Writer + Verifier + Linter) when seven is too much, and the full seven when volume warrants it. The trade-offs and the migration path are documented in [roadmap/README.md](../roadmap/README.md#implementation-paths-graduated-start) — they belong with the implementation timeline, not the profile contracts.

## Anti-patterns

These are the patterns to avoid, drawn from observation of single-agent systems:

- **Profile bleed.** A Librarian writes synthesis. A Writer ingests new sources. Each profile blurs into the next. Symptom: ambiguous responsibility for quality issues.
- **Silent canonization.** A worker says "I'm done"; the system treats that as approval. Symptom: drift between what's marked approved and what was actually reviewed.
- **Re-introducing an orchestrator.** The temptation to add "a profile that decides where work goes" arises when routing rules feel complex. Resist it — encode the rule in the lane-override file or the dispatch policy, not in a reasoning agent. Once a reasoning orchestrator exists, every routing decision becomes hard to audit.
- **Tooling spread.** Every profile gets every MCP. Symptom: profiles can do work outside their lane; debugging becomes unreliable.
- **Linter as auto-fixer.** The Linter silently corrects orphans, retags notes, moves things to archive. Symptom: vault state diverges from human intent without notice.
- **Socratic with write access.** Even a tiny write scope on Socratic ("just to scratch") defeats the architectural protection. Socratic's `policy.allow.write: []` is load-bearing.

The structural guard against all of these is in the permission matrix and the board states. Treat them as non-negotiable.

<!-- memoria-nav -->

---

[← Previous: Board schema and handoff (reference)](../board/card-schema.md)

[Next: Profile commands (reference) →](profile-commands.md)
