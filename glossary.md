---
mode: reference
audience: anyone
topic: general
---

# Glossary

Terms used across the Memoria design docs. Pulled from the README to keep the doc-map terse and to give the vocabulary a dedicated reference home.

## System and architecture

- **Memoria** — the whole system.
- **Hermes** — the agent layer (specialized profiles execute work).
- **Profile** — a Hermes role with bounded permissions, commands, skills, and tools. There are seven: Librarian, Mapper, Socratic, Writer, Verifier, Coder, Linter. Notably absent: no Orchestrator (routing lives in lane-overrides + Kanban dispatch) and no Reviewer (review gates live in the policy MCP and the human).
- **Lane** — a specialist execution path on the board. Each lane maps to a primary profile, but the two don't always share a name: the **Library** lane is run by the **Librarian** profile, **Mapping** by **Mapper**, **Verify** by **Verifier** (Socratic, Writer, Coder, and Linter match). Profiles own outcomes; lanes own claimability — see [profiles/README.md](profiles/README.md#core-design-rule).
- **Vault** — the Obsidian folder tree where durable knowledge lives, organized by lifecycle stage (`00-meta`, `10-inbox`, `20-sources`, `30-synthesis`, `40-workbench`, `50-deliverables`, `90-assets`, `95-archive`).
- **ACP** (Agent Client Protocol) — the editor-level protocol that exposes Hermes profiles to IDE and editor agent panes (Obsidian, VS Code, Zed), giving the human a conversational interface to Hermes from inside the editor. Distinct from the Obsidian Local REST API: the REST API gives Hermes *vault-level* read/write access; ACP gives the human *session-level* chat access to a profile. The ACP connection is served by a local process (`hermes-acp`). See [architecture/capability-stack.md](architecture/capability-stack.md).
- **Human** — the person who owns and runs the vault, making all approval, triage, and promotion decisions: sets `review_status` to `approved`, issues verdicts, promotes notes through lifecycle stages, and configures lane-override policies. Single-user by design.
- **Canonical** (adjective) — approved as the authoritative, immutable form until explicitly archived. Applied consistently across the system: the *source of truth* is the source of truth for a given record type (Zotero for bibliographic metadata; vault for synthesis); *canonical promotion* is the human-approved state transition that moves work from draft to durable; the *canonical verdict* is the three-value set (`approve` / `reject` / `escalate`) every review must resolve to; a *canonical zone* is a vault folder (`30-synthesis/01-claims/`, `30-synthesis/02-reference/`, `30-synthesis/03-moc/`, `50-deliverables/`) where the policy MCP degrades all agent writes to `dry_run` regardless of lane policy — nothing there changes without explicit human approval; `canonical_target` is the card `metadata` field recording which vault path an output should land at if the card is approved. Agent recommendations are necessary but not sufficient to make something canonical — human approval is always the final gate.

## Board and cards

- **Card** — a task on the board with `status`, `assignee`, blocker `reason`, retries, and a handoff summary. The board is the Hermes built-in Kanban (`kanban.db`); see [board/card-schema.md](board/card-schema.md).
- **Handoff payload** — the JSON the worker passes via `kanban_complete`, stored in the card `metadata`: `task_id`, profiles, goal, context, allowed_paths, expected_outputs, review_checks. Self-contained so the receiving worker needs nothing else to start. (Earlier drafts called this the *task packet*.)
- **Review gate** — the `metadata.review_status` overlay that blocks promotion until a human approves. Hermes has no native review state; Memoria layers one on top of `status: done`.
- **Escalation threshold** — the `max_retries` ceiling (default `3`) at which a card auto-moves to `blocked` with `reason: "retry_threshold_exceeded"`. Prevents brittle prompts from burning through retries overnight. Per-lane configurable.
- **Dispatcher** — the Hermes built-in Kanban component that polls the board queue (every 60 seconds — the mandated Memoria cadence, not a per-deployment default; see [board/README.md](board/README.md#dispatch-interval)), claims `ready` cards matching an available lane worker, and advances them to `running`. Ignores `triage` cards (human-only state); automatically re-attempts recoverable failures by returning the card to `ready`. Does not make quality or approval decisions — those remain human actions.
- **`outcome`** — the terminal result recorded on a rejected card: `discarded` (work abandoned) or `superseded` (a revision card spawns and the original is archived permanently). The original card never reopens; revisions live on a new card with provenance back to the original.
- **`verdict`** — the review *decision* on a card: `approve`, `reject`, or `escalate`. The human issues it (Verifier and Linter may attach recommendation-only verdicts in `metadata.agent_verdict`); it drives the next state transition. Distinct from `outcome` (the recorded terminal disposition of an archived card) and the Linter's **verdict band** (PASS / REVIEW / FAIL).
- **`reason`** — freetext field on a card explaining why it is `blocked`. Required whenever `status` is `blocked`. Populated by the dispatcher when the escalation threshold is reached, or by a worker (`kanban_block`) that encounters a decision it cannot make.

## Profile management and configuration

- **Profile source** — under direct profile management, the seven hand-authored profile directories under `.memoria/profiles/memoria-<name>/` in the starter vault. The installer (`install.ps1`) copies these verbatim (with `{{VAULT_PATH}}` substitution in `mcp.json`) into `~/.hermes/profiles/memoria-<name>/`. The deferred [profile-compilation design](roadmap/profile-compilation.md) describes the alternative compile-time-merge model that Memoria does not currently use.
- **`.env.EXAMPLE`** — per-profile manifest of required and optional env vars, commented out, shipped with every profile. Human copies to `.env` and fills in. Excluded from `hermes profile install`/`update` so secrets never travel between machines.
- **Automation tier** — the system-wide autonomy *setting* in profile config: `strict` (default; propose-only), `standard` (safe auto-fixes + low-stakes triage), or `minimal` (also scheduled answer drafting). Each tier is an explicit unlock from the previous. Controls how much the system acts without human confirmation — analogous to SAE L0–L5 autonomy levels, applied to knowledge-work tasks. **Distinct from two other autonomy scales** the docs also use: the per-profile **autonomy Level** (1–3, an *invocation-cadence* label — see [profiles/README.md](profiles/README.md#autonomy-levels)) and Chen 2026's **L1–L5** field taxonomy (Memoria's position overall — see [vision.md](vision.md#position-on-the-autonomy-spectrum)). In short: *tier* = how much the system may act; *Level* = how a profile is invoked; *L-level* = where Memoria sits in the field.

## Policy MCP and audit

- **Policy MCP** — the runtime write-gate. Intercepts every vault write, checks the lane-override rules, returns `allow` / `allow_with_log` / `deny` / `dry_run`, and appends every operationally significant decision to the audit log with SHA-256 hashes for tamper detection. Source in `.memoria/mcp/` (Python).
- **Lane-override file** — per-lane YAML at `.memoria/lane-overrides/{lane}.yaml` declaring `policy.allow`, `policy.deny`, `policy.require`, and `routing`. Read by the policy MCP at startup; the machine-readable counterpart to the folder permission matrix.
- **Audit log** — append-only JSONL trail of every policy-MCP decision at `00-meta/02-logs/audit.jsonl`. Feeds the `audit-log` dashboard.
- **Skill-conditional policy** — mechanism by which a skill's SKILL.md frontmatter declares additive `policy.deny` rules that stack on top of the host lane's policy for the session. Session inherits the narrower of (lane policy ∩ skill policy); cannot be loosened from inside the session. Mechanism in [policy-mcp.md](architecture/policy-mcp.md#skill-conditional-policy); 01-architecture has a brief summary.
- **Restrictive skill** — a skill that uses skill-conditional policy to tighten the host lane. The only currently-shipped restrictive skill is `counter-outline` (Writer-loaded; scratch-only writes to `40-workbench/01-projects/*/framing/`). The capabilities that used to be restrictive skills (`socratic-processing`, `lens-reading`) are now native to the dedicated Socratic profile, whose lane policy is `policy.allow.write: []` — profile-level enforcement is stricter than skill-level. See [profiles/README.md](profiles/README.md#skills-with-restrictive-policy).
- **Fail-closed startup** — the Hermes API requires its auth token (`API_SERVER_KEY`) on every bind, including the default loopback bind, and a non-loopback bind must be explicitly configured (`API_SERVER_HOST`). Closes the always-on option's largest attack surface.

## Observability and verdicts

- **Trust score** — 0–100 aggregate per lane on the [fleet-health dashboard](dashboards/fleet-health.md), combining audit deny rate, drift incidents, secret hits, retry rate, success rate, and (for lanes that produce human-approvable suggestions) accept/reject ratios on inline `[!suggestions]` callouts. Bands: 90+ healthy, 70–89 watch, <70 act. Ratio sub-thresholds: >90% accept = rubber-stamping (down-weight the lane's score); <20% accept = prompt drift (down-weight the lane's score).
- **Verdict band** — the Linter's rollup of findings into PASS / REVIEW / FAIL, gating scheduled work. Parallel structural concept to the trust score.
- **Structural detector** — the Linter's eight deterministic, zero-LLM drift checks, each named by a descriptive slug (legacy `M1`–`M8` codes shown in parentheses for transition): `profile-install-drift` (M1; vault source vs deployed copy under `~/.hermes/profiles/`), `vault-hash-drift` (M2; non-MCP write / audit-log SHA-chain break), `skeleton-drift` (M3), `dashboard-field-drift` (M4), `command-vocab-drift` (M5), `plugin-config-drift` (M6; working `.obsidian/plugins/<plugin>/data.json` vs git HEAD), `orphan-working-files` (M7; `.tmp.*`, `.bak`, editor backups outside transient zones), `extract-path-broken` (M8; paper-note's `extract_path` points to a missing file). See `.memoria/profiles/memoria-linter/M-detectors.md` in the starter vault.
- **Graceful degradation** — dashboard discipline for handling a missing query dependency (a frontmatter field, an aggregator note, an unresolved design decision): an empty result paired with a markdown explanation, not a stack trace.

## Surfaces

- **Surface** (noun) — any place where Memoria state appears or a Memoria action is taken. The docs cut this two ways: the **four surface *types*** inside Obsidian (persistent / modal / inline / ambient — see [surfaces/README.md](surfaces/README.md)) and the **five human-facing *channels*** (Obsidian dashboards + ACP, command palette, CLI, Telegram, API — see [architecture/README.md](architecture/README.md#human-surfaces)). The verb *to surface* (bring something to attention) is a separate, unrelated use.
- **Workspace layout** — saved Obsidian workspace (Human / Reading / Drafting) bound to `Cmd-1` / `Cmd-2` / `Cmd-3`. One layout per cognitive mode; mode shifts become muscle memory. See [surfaces/modal.md](surfaces/modal.md).
- **Inline callout surface** — agent output rendered in-place inside a note rather than on a dashboard: `[!brief]` (Mapper's comparative read on paper notes), `[!suggestions]` (Librarian's link candidates pending approval), `[!verification]` (Verifier's claim trace on drafts). Requires the Callout Manager plugin. See [surfaces/inline.md](surfaces/inline.md).
- **Mock REST API** — a fixture-backed HTTP server implementing the same endpoints as the live Obsidian Local REST API, used for CI testing the board layer without live Obsidian. See [roadmap/timeline.md Phase 4](roadmap/timeline.md#phase-4--kanban-board-state-machine).

## Deployment

- **Deployment option** — human's choice of how the vault and execution layer sync across machines. Four patterns: **`local-only`** (single workstation), **`local-mesh`** (Syncthing peer-to-peer with no VPS — desktop + laptop), **`obsidian-sync`** (cloud-managed), **`always-on`** (Syncthing + VPS, always-on agent). See [roadmap/deployment-options.md](roadmap/deployment-options.md).

## Computational methods

- **Method class** — categorization of a task by computational approach. **Deterministic** (single right answer, derivable via regex / graph / similarity / classifier), **hybrid** (deterministic narrowing + LLM enrichment on the residual), or **generative** (open-ended composition, LLM-required). Memoria's design names this boundary explicitly to avoid using LLMs for tasks where regex or vector similarity would be correct, cheaper, faster, and more testable. See [architecture/why-computational-methods.md](architecture/why-computational-methods.md).
- **Hybrid pattern** — the most common Memoria pattern: a deterministic step reduces N candidates to K (where K ≪ N), then an LLM step generates or judges over the K. Used by `cite-check`, `[!brief]`, `[!suggestions]`, and `_proposed_classification`. The deterministic step's output is the audit trail; the LLM's output is the surface presentation. See [architecture/why-computational-methods.md](architecture/why-computational-methods.md#the-hybrid-pattern).
- **Classifier-with-LLM-fallback** — specific instance of the hybrid pattern. A small multi-label classifier trained on human's past classification decisions proposes labels with a calibrated softmax confidence; above the confidence threshold (~0.85), accept directly; below, fall back to an LLM proposal. Calibration improves as the corpus grows and retrains. Used for `_proposed_classification` ([profiles/librarian.md](profiles/librarian.md)). Resolves [Decision 11](decisions/README.md) (resolved 2026-05).
- **Model routing** — the discipline of sending synthesis to Claude and bulk / mechanical tasks (embed, classify, quick-summary) to cheaper models via OpenRouter or similar. Configured per profile in `config.yaml`. See [architecture/capability-stack.md](architecture/capability-stack.md#model-routing-synthesis-on-claude-cheap-tasks-elsewhere).

## Pipeline stages

- **Discuss stage** — upstream pipeline stage between classify and distill. The paper note is read and discussed through the **Socratic profile** (write-denied across the entire vault) running its `socratic-processing` command before any claim note is written. Made a stage so the queue of "thought about but not yet written" becomes board-visible. See [workflows/upstream/discuss.md](workflows/upstream/discuss.md).
- **Assess stage** — first downstream pipeline stage. Mapper runs `scope-project` to produce a corpus map for a project (`40-workbench/01-projects/*/map/corpus-map.md`); the human then decides whether the corpus is ready to write or needs more reading first. See [workflows/downstream/assess.md](workflows/downstream/assess.md).
- **Frame stage** — second downstream stage. Writer generates 2–3 competing project framings via `counter-outline` (scratch-only writes); Socratic optionally produces lens-based framings via `lens-reading` (conversational, write-denied). Outputs land scratch-only in `40-workbench/01-projects/*/framing/`. Human commits to one via `framing/CHOSEN.md`. Prevents "first outline wins by default." See [workflows/downstream/frame.md](workflows/downstream/frame.md).
- **Verify stage** — downstream stage after draft. Verifier runs `cite-check` to trace every substantive claim back to a claim note; failed traces spawn `gap:` cards in the upstream queue, closing the gap-loop into discovery. Output is a `[!verification]` callout at the top of the draft. See [workflows/downstream/verify.md](workflows/downstream/verify.md).
- **Revise stage** — downstream stage between verify and export. Human addresses verification findings — soften the claim, pursue the gap, or accept-soft — until the verify→revise loop closes. Export is blocked on revise's closure. See [workflows/downstream/revise.md](workflows/downstream/revise.md).

## Note types

- **Source** (umbrella term, *not* a note type) — anything under `20-sources/`. Two ontological kinds: **items** (*things*, described by their properties — includes papers and other `item-note`s) and **entities** (*actors*, described by their relationships — people, organizations, venues). "Source" is never a `type:` value; the concrete types are `paper-note`, `item-note`, `person-note`, `organization-note`, `venue-note`.
- **Paper note** (`paper-note`) — one note per citable paper (journal article, conference paper, preprint, **report**, thesis); lives in `20-sources/01-papers/`. A paper is a *specialized kind of item* with extra properties (citekey, DOI, authors, venue, `pub_status`); it earns its own folder and type for volume and workflow reasons. **Datasets and software are `item-note`s, not paper notes.**
- **Claim note** (`claim-note`) — a single durable idea, in your own words; lives in `30-synthesis/01-claims/`.
- **Reference note** (`reference-note`) — a stable reference page promoted from the claim layer; lives in `30-synthesis/02-reference/`.
- **MOC** — Map of Content, a navigational hub; lives in `30-synthesis/03-moc/`.
- **Fleeting note** — raw capture awaiting promotion or discard; lives in `10-inbox/01-fleeting/`.
- **Answer note** (`answer-note`) — an agent-drafted, cited answer to a research question; lives in `10-inbox/02-answers/`. An unreviewed inbox draft awaiting human approval; promoted to a `claim-note` if accepted, not durable knowledge itself.
- **Deliverable** — finished manuscript, presentation, media asset, or release; lives in `50-deliverables/`.

## Future-direction terms

- **Propagation debts** — queue of dependents that need re-evaluation when a high-traffic note changes (a `claim-note` promoted to `evergreen`, a `reference-note` updated). Materialized by the Linter into a dashboard, worked down by the human.
- **Execution-trace reflection** — retry pattern where the Kanban dispatcher (or a dedicated reflection skill) reads a failure trace and synthesizes a *modified* handoff payload for the next attempt, rather than re-dispatching the same payload. Strictly per-card scoped; never rewrites prompts or skills.

<!-- memoria-nav -->

---

[← Previous: Tutorial: Set up Memoria from zero](tutorials/01-set-up-from-zero.md)
