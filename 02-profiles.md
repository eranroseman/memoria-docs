# 02 — Hermes profiles

Hermes is the worker layer. It is split into seven specialist profiles. Each profile has a focused mission, narrow folder permissions, a small command surface, the skills it actually needs, and an explicit set of MCPs / external tools. Profiles never share permissions by default — every privilege is granted, not inherited.

This document is the canonical reference for what each profile can do, what it cannot do, and how it relates to the others.

## The seven profiles

| Profile | Primary lane | Autonomy level | One-line mission |
| --- | --- | --- | --- |
| **Researcher** | Research | Level 1–2 (background + Kanban) | Discover, ingest, enrich, and classify evidence. Optimistic. |
| **Cartographer** | Cartography | Level 2 (Kanban-pulled) | Map the corpus for a project: scope reports, gap reports, cluster density. Read-only across vault. |
| **Socratic** | Socratic | Level 3 (interactive) | Sharpen the operator's thinking through questioning and lens-based reading. Architecturally write-denied. |
| **Writer** | Writer | Level 2 with review gate | Turn evidence into drafts, synthesis notes, and wiki-ready prose. |
| **Verifier** | Verify | Level 2 (Kanban-pulled) | Trace claims to sources; verify citations; flag duplicates and retractions. Read-only across vault except verification reports. |
| **Coder** | Coder | Level 2 (external dispatch) | Build and maintain code artifacts, scripts, repos. Transactional. |
| **Linter** | Linter | Level 1 (scheduled) | Validate structure, metadata, schema, link health. Default dry-run. Also owns session-log and audit-trail housekeeping. |

**No Orchestrator.** Routing decisions live in lane-override files and Kanban dispatch rules, not in a reasoning agent. When a card is created, its lane field determines which profiles can claim it; the dispatch rules decide which one does. There is no profile whose job is to "decide where work goes" — that decision is encoded in policy, not delegated to a worker.

**No Reviewer.** Review gates are board states (`awaiting-review`, `approved`) enforced by the policy MCP and resolved by the operator directly. The mechanical parts of what Reviewer used to do (claim tracing, similarity checks, retraction checks) live in Verifier. The judgment parts (approve / reject) live with the human, surfaced through inline `[!suggestions]` and `[!verification]` callouts.

## Lane permissions matrix

The **consolidated lane view** — what each lane may run (skills), what tools it uses, what's denied, and where it can write. It's the human-readable summary you check first when a worker behaves unexpectedly. A complementary [folder × profile view](#folder-permission-matrix) appears below; the YAML form the policy MCP actually reads is in [lane-override files](#lane-override-files).

| Lane | Primary role | Core commands | Allowed skills | Allowed tools | Denied capabilities | Write scope |
| --- | --- | --- | --- | --- | --- | --- |
| **Research** | Discover and ingest evidence | `discover`, `ingest`, `enrich`, `triage`, `query` | `paper-lookup`, `arxiv-search`, `pyzotero`, `citation-management`, `literature-review`, `obsidian-paper-note`, `generic-rest-bridge` | `search_web`, `fetch_url`, vault read/write | canonical publish, destructive shell, unrestricted HTTP | `10-inbox/`, `20-sources/` |
| **Cartography** | Map the corpus | `scope-project`, `gap-report`, `cluster-map`, `comparative-brief` | `scope-project`, `gap-report`, `cluster-cartography`, `comparative-brief` | vault read | all write tools, external APIs, drafting | `40-workbench/01-projects/*/corpus-map.md`, `*/gap-report.md`, `*/comparative-briefs/*` |
| **Socratic** | Question without producing | `socratic-processing`, `lens-reading` | `socratic-processing`, `lens-reading` (parameterized: `mamykina-lens`, `veinot-equity-lens`, etc.) | vault read | **all write tools** (`policy.allow.write: []`), external APIs, drafting, queue dispatch (`routing.invocation: interactive_only`) | (none — `read_only_mode`, hard) |
| **Writer** | Draft and synthesize | `draft`, `query`, `lint`, `promote` (handoff) | `llm-wiki draft`, `note-refactor`, `scientific-writing`, `counter-outline` | `search_web`, `fetch_url`, vault read/write | `generic-rest-bridge`, external-API skills, publish-canonical | `10-inbox/02-synthesis/`, `40-workbench/02-drafts/`, `40-workbench/01-projects/*/framing/` (via `counter-outline`) |
| **Verify** | Verify claims, citations, duplicates | `cite-check`, `similarity-check`, `find-duplicates`, `retraction-check` | `cite-check`, `similarity-check`, `find-duplicates`, `retraction-check` | `search_web`, `fetch_url`, vault read | all write tools except verification reports, drafting | `40-workbench/01-projects/*/verification/*` |
| **Linter** | Validate, report, and log | `lint`, `schema-check`, `schema-migrate`, `health-report`, `graph-analyze`, `session-log`, `dry-run`, `report` | `schema-check`, `graph-analyze`, `health-report`, `session-log` | vault read, file indexer, Git | canonical edits, schema-content auto-fixes, work spawning | `00-meta/04-logs/` (audit and session logs only), dry-run reports |
| **Coder** | Code artifacts | `code`, `commit`, `revert`, `workspace`, `scaffold` | `scaffold-code-note`, `workspace-coordinate`, `commit-and-document` (thin Hermes-side wrappers; substantive coding work lives in the external coding agent — see [Coder ↔ external coding agent](#coder--external-coding-agent)) | Git, filesystem, repo APIs | canonical edits, prose ownership | `40-workbench/03-code/` |

Rules of thumb:

- **Networked skills are lane-restricted.** Only Research can call `generic-rest-bridge`. The bridge is the escape hatch (see [01-architecture/capability-stack.md](01-architecture/capability-stack.md#generic-rest-bridge--the-escape-hatch)); confining it to one lane keeps external I/O auditable.
- **Socratic and Cartographer are read-only.** Any `decision: allow` for `profile: memoria-socratic` or `profile: memoria-cartographer` on a `write` action outside its declared scratch path is a configuration bug — surfaced by the [audit-log dashboard](dashboards/audit-log.md).
- **No lane writes to canonical zones.** `30-synthesis/01-permanent/`, `30-synthesis/03-moc/`, and `50-deliverables/` are policy-MCP `dry_run` for every lane. Promotion is always synchronous with human attention.

## Autonomy levels

Each profile's autonomy level is part of its contract:

- **Level 1 (background)** — runs unattended on a cron schedule. Produces reports; never acts on canonical content. Linter is the canonical example.
- **Level 2 (Kanban-pulled)** — picks up cards from its lane queue. Produces output to review-gated paths. The bulk of Memoria's work.
- **Level 2 with review gate** — produces drafts that don't promote to canonical without explicit human approval. Writer.
- **Level 3 (interactive)** — invoked synchronously by the operator (via ACP, command palette, or Telegram). No queue. Socratic.
- **Level 2 (external dispatch)** — handoffs to an external agent (Claude Code, Aider, Codex) via task packets. Coder.

The level governs both the cadence (background / kanban-pulled / interactive) and the expected output mode (report / artifact / conversation). It does not loosen any policy MCP rule.

## Profile temperament

Different profiles have different default postures:

- **Researcher: optimistic.** Errs toward including candidates and proposing classifications.
- **Cartographer: read-only.** Summarizes what exists; never proposes new sources or claims. Coexists with the [Smart Connections](operations/obsidian-plugins/smart-connections.md) plugin as a complement — Cartographer's outputs (`corpus-map.md`, `gap-report.md`) are deterministic and reviewable; Smart Connections' suggestions are statistical and opaque. Memoria's design depends on neither, and the two don't compete for the same artifact.
- **Socratic: question-only.** Cannot write to the vault under any circumstance. The lane policy is `policy.allow.write: []`.
- **Writer: ownership-sensitive.** Delegates lookup and structure checks, but owns the synthesis. Output is review-gated.
- **Verifier: mechanical and conservative.** Treats unverifiable claims as a reason to flag, not guess.
- **Coder: transactional.** Per-task commits, scoped repo changes, no spillover into note taxonomy.
- **Linter: dry-run by default.** Never silently fixes canonical content. Reports issues; humans decide on fixes.

## Folder permission matrix

This is the operational access map. For the layered folder structure these columns refer to, see [05-notes-folders.md](05-notes-folders.md).

| Profile | 00-meta | 10-inbox | 20-sources | 30-synthesis/01-permanent | 30-synthesis/02-wiki | 30-synthesis/03-moc | 40-workbench | 50-deliverables |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Researcher | Read | Write (discovery, candidates) | Write (create, enrich) | Read only | Read for context | Read for context | Read | Read only |
| Cartographer | Read | Read only | Read only | Read only | Read only | Read only | Write (`01-projects/*/corpus-map.md`, `*/gap-report.md`) | Read only |
| Socratic | Read | Read only | Read only | Read only | Read only | Read only | Read only | Read only |
| Writer | Read | Write (synthesis drafts) | Read only | Read only | Write drafts (review-gated) | Read; suggest | Write (drafts, framing) | Read only unless export task |
| Verifier | Read | Read only | Read only | Read only | Read only | Read only | Write (`01-projects/*/verification/*`) | Read only |
| Coder | Read | Read only | Read only | Read only | Read for context | Read | Write (`40-workbench/03-code/`) | Read / write on explicit export tasks |
| Linter | Read; write `04-logs/` (audit, session) | Read | Read | Read | Read | Read | Read | Read |

Rule of thumb: **canonical synthesis remains human-owned** in `30-synthesis/01-permanent/`. **Schema governance remains human-owned** in `00-meta/` except for the logs subfolder Linter writes to. Project scratch (`40-workbench/01-projects/`) is the only zone where multiple profiles write, and each profile writes to its own named subfolder.

## Lane-override files

The folder permission matrix above is the human-readable summary. The machine-readable encoding lives in **lane-override files** at `.memoria/lane-overrides/{lane}.yaml`. One file per lane. The policy MCP reads these at startup and enforces them at every write (see [reference/policy-mcp.md](reference/policy-mcp.md)).

### File shape

Each lane-override has the same four blocks. Example for the research lane:

```yaml
lane: research
profile: memoria-researcher

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
  write_scope:                               # the lane's canonical short-list
    - "10-inbox/"
    - "20-sources/"
```

The four blocks:

- **`policy.allow`** — skills the lane may load and paths it may write. Patterns are glob-style relative to the vault root.
- **`policy.deny`** — explicit blocks. Deny wins over allow.
- **`policy.require`** — invariants the worker must honor. Common values: `audit_log` (every action logged), `timeout_required` (no unbounded calls), `source_tracking` (writes carry a provenance trail), `read_only_mode` (no writes at all — used by Socratic and Cartographer), `canonical_write_gate` (writes to canonical zones always degrade to `dry_run`).
- **`routing.write_scope`** — the canonical short-list, usually narrower than `policy.allow.write`. Used by the dispatch rules to decide where a worker's output should land by default.
- **`routing.invocation`** — how the lane is invoked. `dispatched` (default; the Kanban dispatcher pulls cards from this lane's queue) or `interactive_only` (the lane is never queue-dispatched; only synchronous operator invocation reaches it). Socratic uses `interactive_only` so a misconfigured cron job can't queue work onto a write-denied conversational profile.

### One file per lane

| Lane | File | Notes |
| --- | --- | --- |
| Research | `.memoria/lane-overrides/research.yaml` | May call external APIs through approved skills. |
| Cartography | `.memoria/lane-overrides/cartographer.yaml` | `read_only_mode` for the vault; project-scratch writes only. |
| Socratic | `.memoria/lane-overrides/socratic.yaml` | Hard `policy.allow.write: []` — write-denied across the vault. `routing.invocation: interactive_only` — never queue-dispatched. |
| Writer | `.memoria/lane-overrides/writer.yaml` | May draft and refine; no direct external API; canonical writes degrade to `dry_run`. |
| Verify | `.memoria/lane-overrides/verifier.yaml` | `read_only_mode` for the vault; verification-report writes only. |
| Coder | `.memoria/lane-overrides/coder.yaml` | Writes to `40-workbench/03-code/` only. |
| Linter | `.memoria/lane-overrides/linter.yaml` | Dry-run by default; `auto_fix` is class-gated by the policy MCP; owns `00-meta/04-logs/` for session and audit-trail housekeeping. |

### Why not encode this in `config.yaml`

The profile-level `config.yaml` says what the profile *is* — model, command surface, skill registry, base MCPs. The lane-override says what the profile *may do in this vault*. The split lets the same profile run against multiple vaults with different policies, and it lets us review permissions independently of model and tooling.

### Composability

Two files compose at session start:

1. `~/.hermes/profiles/memoria-<profile>/config.yaml` — model, command surface, skill registry, base MCPs.
2. `.memoria/lane-overrides/<lane>.yaml` — policy and routing for this vault.

If a worker would claim a card outside its lane override, the policy MCP returns `deny`. If a worker tries to write into a canonical zone, the MCP returns `dry_run` and the action becomes a board comment for the human to act on. Either way, nothing canonical changes without an explicit approval step.

## Skill governance

Skills are first-class objects with their own state machine (`intake → proposed → scaffolded → testing → needs-review → approved → active → archived`), parallel to the card lifecycle. New skills enter the system through a 7-step onboarding checklist that ends with adding the skill to a lane-override `policy.allow.skills` list. The state machine, registry format (`00-meta/07-skills/`), onboarding checklist, and the bridge-to-dedicated-skill graduation thresholds live in [07-roadmap.md](07-roadmap/skill-governance.md) — they are an operational discipline that matters once the system is accumulating tooling, not when first configuring profiles.

### Skills with restrictive policy

Most skills inherit their host lane's policy unchanged. A small set of skills *tighten* it further by declaring additive `policy.deny` rules in their SKILL.md frontmatter. The mechanism (composition semantics, enforcement path, frontmatter contract) lives in [reference/policy-mcp.md](reference/policy-mcp.md#skill-conditional-policy); the **catalog** of Memoria's restrictive skills lives here:

| Skill | Loaded by | What it adds | Why |
| --- | --- | --- | --- |
| `counter-outline` | Writer | `deny: write:30-synthesis/**`, `deny: write:50-deliverables/**`, `deny: write:10-inbox/**` — scratch-only | Generates 2–3 competing project outlines before drafting begins. Outputs land in `40-workbench/01-projects/*/framing/` only; "first outline wins by default" is the failure mode this prevents. Loaded only when Writer is in the Frame stage of a downstream pipeline. |

The shared property: a skill's deny rules are stricter than the host lane's. A `counter-outline` session under Writer is more restricted than Writer's baseline, never more permissive — and the narrowing is enforced at the policy MCP, not by prompt discipline.

**Note on what's NOT here.** Earlier versions of this design had `socratic-processing` and `lens-reading` as restrictive skills. They've been promoted to a profile (Socratic) whose lane policy is `policy.allow.write: []` natively. Profile-level enforcement is stricter than skill-level enforcement because it can't be sidestepped by loading the host profile without the skill. The skill-conditional-policy mechanism is preserved for `counter-outline` and any future scratch-only skills that are too narrow to justify their own profile.

## Per-profile contracts

Each profile's operational contract — exact allowed folders, commands, skills, tooling, rules, and exit conditions — lives in its own AGENTS.md file. These are the runtime contracts that Hermes reads at session start; the design documents summarize, the AGENTS.md files specify.

| Profile | Design summary | Runtime contract (SOUL.md, in starter vault) |
| --- | --- | --- |
| Researcher | [profiles/researcher.md](profiles/researcher.md) | `.memoria/profiles/memoria-researcher/SOUL.md` |
| Cartographer | [profiles/cartographer.md](profiles/cartographer.md) | `.memoria/profiles/memoria-cartographer/SOUL.md` |
| Socratic | [profiles/socratic.md](profiles/socratic.md) | `.memoria/profiles/memoria-socratic/SOUL.md` |
| Writer | [profiles/writer.md](profiles/writer.md) | `.memoria/profiles/memoria-writer/SOUL.md` |
| Verifier | [profiles/verifier.md](profiles/verifier.md) | `.memoria/profiles/memoria-verifier/SOUL.md` |
| Coder | [profiles/coder.md](profiles/coder.md) | `.memoria/profiles/memoria-coder/SOUL.md` |
| Linter | [profiles/linter.md](profiles/linter.md) | `.memoria/profiles/memoria-linter/SOUL.md` (plus `M-detectors.md`) |

The tables in this document — folder permission matrix, command summary, lane mapping, delegation ladder, workflow ownership — are the architectural *summary*. For the specific permission lists, dry-run defaults, and exit conditions per profile, read the contract.

## Coder ↔ external coding agent

The coder profile is **narrow by design**: it scaffolds `code-note` handoffs and documents provenance. The actual editing happens in a specialized external coding agent (Kilocode, Aider, Claude Code, Codex) running as a peer with a shared filesystem — the vault is its read-only context, `40-workbench/03-code/` is its write zone, and the human reviews `code-note` updates as the review gate. No orchestration infrastructure is needed: the two agents coordinate through the markdown handoff, not through subprocess dispatch.

**Full mechanism** — the two-agent boundary, VS Code workspace JSON for "small scripts in the vault" vs "real projects in their own repo," the `files.readonlyInclude` boundary, agent-instruction-file placement (`CLAUDE.md` / `.kilocode/rules/` / `.cursorrules`), the rule of thumb for when a project earns its own repo, and the `repo:` frontmatter convention on the `code-note` — lives in [rationale/coder-external-agent.md](rationale/coder-external-agent.md). The same pattern generalizes to **external rendering agents** (open-design) for visual deliverables, with `50-deliverables/` as the write zone and the `deliverable` note as the handoff artifact.

## Command reference

The Core commands column in the [Lane permissions matrix](#lane-permissions-matrix) lists the verbs each profile uses; per-profile skill and tool surfaces are documented in the same matrix and in the per-profile [profiles/*.md](profiles/) contracts. Beyond the core verbs, this is the full operational command catalog with dry-run defaults and owner profiles. Many of these are also described inline in [04-workflows.md](04-workflows.md).

| Command | Purpose | Primary profile | Dry-run? |
| --- | --- | --- | --- |
| `ingest` | Create the right note type in the right folder with enrichment. | Researcher | No (creates notes) |
| `discover` | Forward / backward citation search or concept-driven search. | Researcher | No (writes candidates) |
| `enrich` | Re-run API enrichment on existing notes. | Researcher | No |
| `triage` | Re-propose `_draft_classification` when a note still needs review. | Researcher | No |
| `obsidian-paper-note` | Full ingest pipeline including PDF extraction (via Marker). | Researcher | No |
| `export prior-labels` | Export vault papers as ASReview priors for pre-ingest screening. | Researcher | N/A |
| `scope-project` | Map the corpus for a project: cluster density, recency distribution, source diversity, gap analysis. Writes `corpus-map.md` to the project folder. | Cartographer | No (writes to project scratch only) |
| `gap-report` | Identify thin-coverage topics adjacent to a project brief. Feeds Verifier's gap-card spawning. | Cartographer | No |
| `cluster-map` | Render a density / recency map for an arbitrary topic across the corpus. | Cartographer | No |
| `comparative-brief` | When a new source enters the queue, generate a brief comparing it against existing claims — overlap, contradiction, new constructs. Drives the inline `[!brief]` callout. | Cartographer | No |
| `socratic-processing` | Question-only conversation about a literature note. No writes. | Socratic | N/A (read-only profile) |
| `lens-reading` | Read a note or cluster through a named theoretical lens. Parameterized by lens name. | Socratic | N/A (read-only profile) |
| `draft` | Search the vault and produce a synthesis note for human review. | Writer | No |
| `cite-check` | Verify citations in drafts before export — every citekey resolves to a real source note. | Verifier | Yes (default) |
| `find-duplicates` | Identify semantically similar claim notes for merge review. Retrospective; runs monthly. | Verifier | Yes (always — never auto-merges) |
| `similarity-check` | Point-of-action check: before a new claim note is filed, surface the top 3 most-similar existing notes. Flag at threshold ~0.8; never block. | Verifier | Yes (always — informational, never auto-merges) |
| `retraction-check` | Scan source notes against Zotero retraction alerts and CrossRef. | Verifier | Yes (default) |
| `lint` | Structural health check across the vault. | Linter | Yes (default) |
| `schema-check` | Verify frontmatter against the canonical schema. | Linter | Yes |
| `schema-migrate` | Propose schema changes between versions. **Always dry-run first.** | Linter | Yes (always required first) |
| `graph-analyze` | Knowledge graph health: orphans, hubs, clusters, link density. | Linter | Yes |
| `session-log` | Write per-session log file to `00-meta/04-logs/`. | Linter | N/A |

Rule: any command that writes to canonical folders (`30-synthesis/01-permanent/`, `30-synthesis/03-moc/`, `50-deliverables/`) or runs a migration must default to dry-run. `schema-migrate` in particular must never be run without reviewing the diff first.

## Routing without an Orchestrator

Routing — "which profile picks up this card?" — is rule-encoded, not reasoned. Three mechanisms together:

1. **The card's `lane` field** determines which profiles are eligible to claim it. A card with `lane: cartography` can only be claimed by a worker whose lane-override file declares it on that lane.
2. **The lane-override file's `routing.write_scope`** declares where that worker's output should land by default, so the worker doesn't have to decide.
3. **The Kanban dispatcher** picks the highest-priority eligible card from each lane's queue when a worker becomes available. Priority is set at card creation (cron-triggered work is usually low-priority; operator-initiated is high).

There is no reasoning step. If the rules can't decide (two profiles eligible, or none eligible), the card sits in `ready` until the operator intervenes. This is the design — silent reasoning about routing is exactly what the policy MCP exists to prevent.

When a card needs to move between lanes (e.g., from research to verify after ingest), the *worker that just finished* moves it. The Researcher who completes an ingest moves the card to the `verify` lane's queue if a similarity check is needed; the Writer who commits a draft fires the verify-on-commit hook. No central agent makes these decisions; the workflow does.

## Lane-to-profile mapping

A lane is a specialist execution path on the board. Each lane has a primary profile that owns it.

| Lane | Researcher | Cartographer | Socratic | Writer | Verifier | Coder | Linter |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Research | **Primary** | Read context | No claim | Source context | Filing-time similarity | No claim | Validate |
| Cartography | Source context | **Primary** | No claim | Consumes corpus map | No claim | No claim | Validate scratch shape |
| Socratic | No claim | No claim | **Primary** | No claim (operator switches profiles) | No claim | No claim | No claim |
| Writer | Source context | Consumes corpus map | Prep (via operator switch) | **Primary** | Consumes draft for `cite-check` | No claim | Catch broken links |
| Verify | No claim | No claim | No claim | Produces drafts to verify | **Primary** | No claim | Catch lingering findings |
| Coder | Motivating sources | No claim | No claim | Prose ownership for code-notes | No claim | **Primary** | Code metadata checks |
| Linter | Context only | Context only | Context only | Draft support | Verification trail | Validate code metadata | **Primary** |

Rule: the lane determines the worker type allowed to claim a card; the board determines the card's state; the **lane-override files determine routing**, not any reasoning agent.

## Delegation ladder

Delegation is strongest in profiles that produce derivative artifacts (drafts, code-notes), weakest in profiles whose value depends on independence or non-production.

| Profile | Delegation posture |
| --- | --- |
| Researcher | **Targeted.** Delegates narrow enrichment or source lookups; keeps discovery ownership. |
| Cartographer | **Low.** Delegates only mechanical retrieval (e.g., to qmd for vector search). Keeps the map authorship. |
| Socratic | **None.** Socratic has nothing to delegate — questions and conversations are the whole product. Cannot delegate writing because it can't write. |
| Writer | **Supportive.** Delegates facts or cleanup; keeps synthesis ownership. Final draft remains local. |
| Verifier | **Very low.** Delegation weakens verification independence. Inspects, traces, flags. |
| Coder | **Moderate.** Delegates helper work, formatting, lookup; keeps implementation control. Commits stay per task. Also delegates substantive coding work to the external coding agent via task packets — see the Coder section above. |
| Linter | **Lowest.** Does not spawn work. May request context, but its job is to validate, report, and log. |

Rule: delegate narrow, temporary, low-risk subtasks; never delegate away the role's defining judgment.

## Workflow ownership

Each workflow has a primary profile. The canonical view — workflows × profiles, with primary / support / no-claim distinctions across all 18 workflows — is the [Role × stage matrix in 04-workflows.md](04-workflows.md#role--stage-matrix). Don't duplicate it here; keep one source of truth.

## Inter-profile handoff patterns

The recommended handoff chain for a typical upstream card (source → claim):

```text
Researcher  ──► [triage-pending]  ──► (operator triages)  ──► [process-pending]
                                                                  │
                                       (operator switches to Socratic; conversation; no writes)
                                                                  │
                                       (operator writes claim note in Writer profile, with Verifier doing similarity-check at filing time)
                                                                  │
                                                                  └──► [synthesized]
```

The recommended handoff chain for a typical downstream card (project → deliverable):

```text
Cartographer  ──► [scope-review]  ──► (operator decides project is ready)  ──► Writer with counter-outline
                                                                                          │
                                                                                          └──► [framing-review] ──► (operator chooses framing)
                                                                                                                          │
                                                                                                                          └──► Writer (draft)
                                                                                                                                  │
                                                                                                                                  └──► Verifier (on commit) ──► [revise or export]
```

Linter can act at any point, on any card, with a dry-run report attached as a comment. Socratic is invoked synchronously by the operator and never appears in queue-based handoff chains. The Coder operates in parallel for code artifacts.

## Core design rule

**Profiles own outcomes; lanes own claimability.**

A profile is a durable identity with a domain — researchers discover, cartographers map, writers synthesize, verifiers verify. The profile is responsible for the quality of its outputs.

A lane is a board-level contract about *who can claim a card*. The lane tells the dispatcher which profile class is allowed to move a card forward; the exit state is part of the lane contract.

The two work together:

- If a card is in the research lane, only researcher-class workers may claim it.
- If a card is `awaiting-review`, only the operator may clear it (no Reviewer profile to do so).
- When a worker finishes its slice, it moves the card to the appropriate exit state — it does not mark the card done.

This is what prevents "everything becomes orchestration." Each profile stays accountable for what it ships.

## Scaling

The seven-profile design is canonical Memoria, but the system supports graduated starts: a **mode-based** single Hermes for the simplest setup, a **four-profile** minimal configuration (Researcher + Writer + Verifier + Linter) when seven is too much, and the canonical seven when volume warrants it. The trade-offs and the migration path are documented in [07-roadmap.md](07-roadmap.md#implementation-paths-graduated-start) — they belong with the implementation timeline, not the profile contracts.

## Anti-patterns

These are the patterns to avoid, drawn from observation of single-agent systems:

- **Profile bleed.** A researcher writes synthesis. A writer ingests new sources. Each profile blurs into the next. Symptom: ambiguous responsibility for quality issues.
- **Silent canonization.** A worker says "I'm done"; the system treats that as approval. Symptom: drift between what's marked approved and what was actually reviewed.
- **Re-introducing an orchestrator.** The temptation to add "a profile that decides where work goes" arises when routing rules feel complex. Resist it — encode the rule in the lane-override file or the dispatch policy, not in a reasoning agent. Once a reasoning orchestrator exists, every routing decision becomes hard to audit.
- **Tooling spread.** Every profile gets every MCP. Symptom: profiles can do work outside their lane; debugging becomes unreliable.
- **Linter as auto-fixer.** The linter silently corrects orphans, retags notes, moves things to archive. Symptom: vault state diverges from human intent without notice.
- **Socratic with write access.** Even a tiny write scope on Socratic ("just to scratch") defeats the architectural protection. Socratic's `policy.allow.write: []` is load-bearing.

The structural guard against all of these is in the permission matrix and the board states. Treat them as load-bearing.

## Next

- For board states and the review gate: [03-board.md](03-board.md).
- For the per-profile AGENTS.md contracts (ready to drop into Hermes config): [profiles/](profiles/).
- For how profiles execute the stage-gated pipeline: [04-workflows.md](04-workflows.md).
