# Glossary

Terms used across the Memoria design docs. Pulled from the README to keep the doc-map terse and to give the vocabulary a dedicated reference home.

## System and architecture

- **Memoria** — the whole system.
- **Hermes** — the agent layer (specialized profiles execute work).
- **Profile** — a Hermes role with bounded permissions, commands, skills, and tools. There are seven: researcher, cartographer, socratic, writer, verifier, coder, linter. Notably absent: no Orchestrator (routing lives in lane-overrides + Kanban dispatch) and no Reviewer (review gates live in the policy MCP and the operator).
- **Lane** — a specialist execution path on the board. Each lane maps to a primary profile.
- **Vault** — the Obsidian folder tree where durable knowledge lives, organized by lifecycle stage (`00-meta`, `10-inbox`, `20-sources`, `30-synthesis`, `40-workbench`, `50-deliverables`, `90-assets`, `95-archive`).

## Board and cards

- **Card** — a task on the board with state, assignee, blocker, retries, and handoff notes.
- **Task packet** — the JSON handoff format carrying `task_id`, profiles, goal, context, allowed_paths, expected_outputs, review_checks, and next_state. Self-contained so the receiving worker needs nothing else to start.
- **Review gate** — the explicit board state that blocks promotion until a human approves.
- **Escalation threshold** — the `retry_count` ceiling (default `> 3`) at which a card auto-moves to `blocked-on-human` with `blocked_reason: "retry_threshold_exceeded"`. Prevents brittle prompts from burning through retries overnight. Per-lane configurable.

## Profile management and configuration

- **Profile source** — under direct profile management, the seven hand-authored profile directories under `.memoria/profiles/memoria-<name>/` in the starter vault. The installer (`install.ps1`) copies these verbatim (with `{{VAULT_PATH}}` substitution in `mcp.json`) into `~/.hermes/profiles/memoria-<name>/`. The deferred [profile-compilation design](profile-compilation.md) describes the alternative compile-time-merge model that Memoria does not currently use.
- **`.env.EXAMPLE`** — per-profile manifest of required and optional env vars, commented out, shipped with every profile. Operator copies to `.env` and fills in. Excluded from `hermes profile install`/`update` so secrets never travel between machines.
- **Hook profile** — system-wide automation tier set in profile config: `strict` (default; propose-only), `standard` (safe auto-fixes + low-stakes triage), or `minimal` (also scheduled synthesis drafting). Each tier is an explicit unlock from the previous.

## Policy MCP and audit

- **Policy MCP** — the runtime write-gate. Intercepts every vault write, checks the lane-override rules, returns `allow` / `allow_with_log` / `deny` / `dry_run`, and appends every operationally significant decision to the audit log with SHA-256 hashes for tamper detection. Source in `.memoria/mcp/` (Python).
- **Lane-override file** — per-lane YAML at `.memoria/lane-overrides/{lane}.yaml` declaring `policy.allow`, `policy.deny`, `policy.require`, and `routing`. Read by the policy MCP at startup; the machine-readable counterpart to the folder permission matrix.
- **Audit log** — append-only JSONL trail of every policy-MCP decision at `00-meta/04-logs/audit.jsonl`. Feeds the `audit-log` dashboard.
- **Skill-conditional policy** — mechanism by which a skill's SKILL.md frontmatter declares additive `policy.deny` rules that stack on top of the host lane's policy for the session. Session inherits the narrower of (lane policy ∩ skill policy); cannot be loosened from inside the session. Mechanism in [policy-mcp.md](policy-mcp.md#skill-conditional-policy); 01-architecture has a brief summary.
- **Restrictive skill** — a skill that uses skill-conditional policy to tighten the host lane. The only currently-shipped restrictive skill is `counter-outline` (Writer-loaded; scratch-only writes to `40-workbench/01-projects/*/framing/`). The capabilities that used to be restrictive skills (`socratic-processing`, `lens-reading`) are now native to the dedicated Socratic profile, whose lane policy is `policy.allow.write: []` — profile-level enforcement is stricter than skill-level. See [02-profiles.md](../02-profiles.md#skills-with-restrictive-policy).
- **Fail-closed startup** — the local HTTP bridge defaults to loopback and refuses to bind a non-loopback interface without `HERMES_BRIDGE_TOKEN` set. Closes Option C's largest attack surface.

## Observability and verdicts

- **Trust score** — 0–100 aggregate per lane on the [fleet-observability dashboard](../dashboards/fleet-observability.md), combining audit deny rate, drift incidents, secret hits, retry rate, success rate, and (for lanes that produce human-approvable suggestions) accept/reject ratios on inline `[!suggestions]` callouts. Bands: 90+ healthy, 70–89 watch, <70 act. Ratio sub-thresholds: >90% accept = rubber-stamping (down-weight the lane's score); <20% accept = prompt drift (down-weight the lane's score).
- **Verdict band** — the linter's rollup of findings into PASS / REVIEW / FAIL, gating scheduled work. Parallel structural concept to the trust score.
- **Structural detector** (M1–M8) — the linter's deterministic, zero-LLM drift checks: M1 profile install drift (vault source vs deployed copy under `~/.hermes/profiles/`), M2 vault hash drift, M3 skeleton drift, M4 dashboard field drift, M5 command vocabulary drift, M6 plugin-config drift (working `.obsidian/plugins/<plugin>/data.json` vs git HEAD), M7 orphan working files (`.tmp.*`, `.bak`, editor backups outside transient zones), M8 extract path broken link (source-note's `extract_path` points to a missing file). See `.memoria/profiles/memoria-linter/M-detectors.md` in the starter vault.
- **Capability gate** — dashboard discipline for degrading gracefully when a query's dependency (a frontmatter field, an aggregator note, an unresolved design decision) is missing. Empty result paired with a markdown explanation, not a stack trace.

## Surfaces

- **Workspace layout** — saved Obsidian workspace (Operator / Reading / Drafting) bound to `Cmd-1` / `Cmd-2` / `Cmd-3`. One layout per cognitive mode; mode shifts become muscle memory. See [06-surfaces.md](../06-surfaces/modal.md).
- **Inline callout surface** — agent output rendered in-place inside a note rather than on a dashboard: `[!brief]` (Cartographer's comparative read on literature notes), `[!suggestions]` (Researcher's link candidates pending approval), `[!verification]` (Verifier's claim trace on drafts). Requires the Callout Manager plugin. See [06-surfaces.md](../06-surfaces/inline.md).
- **Mock bridge** — a fixture-backed HTTP server implementing the same endpoints as the real Obsidian bridge, used for CI testing the board layer without live Obsidian. See [07-roadmap/timeline.md Phase 4](../07-roadmap/timeline.md#phase-4--kanban-board-state-machine).

## Deployment

- **Deployment option** — operator's choice of how the vault and execution layer sync across machines. Four patterns: **A** (local only, single workstation), **A+** (local multi-device, Syncthing peer-to-peer with no VPS — desktop + laptop), **B** (Obsidian Sync, cloud-managed), **C** (Syncthing + VPS, always-on agent). See [07-roadmap/deployment-options.md](../07-roadmap/deployment-options.md).

## Computational methods

- **Method class** — categorization of a task by computational approach. **Deterministic** (single right answer, derivable via regex / graph / similarity / classifier), **hybrid** (deterministic narrowing + LLM enrichment on the residual), or **generative** (open-ended composition, LLM-required). Memoria's design names this boundary explicitly to avoid using LLMs for tasks where regex or vector similarity would be correct, cheaper, faster, and more testable. See [rationale/computational-methods.md](../rationale/computational-methods.md).
- **Hybrid pattern** — the most common Memoria pattern: a deterministic step reduces N candidates to K (where K ≪ N), then an LLM step generates or judges over the K. Used by `cite-check`, `[!brief]`, `[!suggestions]`, and `_draft_classification`. The deterministic step's output is the audit trail; the LLM's output is the surface presentation. See [rationale/computational-methods.md](../rationale/computational-methods.md#the-hybrid-pattern).
- **Classifier-with-LLM-fallback** — specific instance of the hybrid pattern. A small multi-label classifier trained on operator's past triage decisions proposes labels with a calibrated softmax confidence; above the confidence threshold (~0.85), accept directly; below, fall back to an LLM proposal. Calibration improves as the corpus grows and retrains. Used for `_draft_classification` ([profiles/researcher.md](../profiles/researcher.md)). Resolves [Decision 11](../07-roadmap/decisions/README.md) (resolved 2026-05).
- **Model routing** — the discipline of sending synthesis to Claude and bulk / mechanical tasks (embed, classify, quick-summary) to cheaper models via OpenRouter or similar. Configured per profile in `config.yaml`. See [01-architecture/capability-stack.md](../01-architecture/capability-stack.md#model-routing-synthesis-on-claude-cheap-tasks-elsewhere).

## Pipeline stages

- **Process stage** — upstream pipeline stage between triage and synthesize. The literature note is read and discussed through the **Socratic profile** (write-denied across the entire vault) running its `socratic-processing` command before any claim note is written. Made a stage so the queue of "thought about but not yet written" becomes board-visible. See [04-workflows/14-process.md](../04-workflows/14-process.md).
- **Scope stage** — first downstream pipeline stage. Cartographer runs `scope-project` to produce a corpus map for a project (`40-workbench/01-projects/*/corpus-map.md`); the operator decides whether the corpus is ready to write or needs more reading first. See [04-workflows/15-scope.md](../04-workflows/15-scope.md).
- **Frame stage** — second downstream stage. Writer generates 2–3 competing project framings via `counter-outline` (scratch-only writes); Socratic optionally produces lens-based framings via `lens-reading` (conversational, write-denied). Outputs land scratch-only in `40-workbench/01-projects/*/framing/`. Operator commits to one via `framing/CHOSEN.md`. Prevents "first outline wins by default." See [04-workflows/16-frame.md](../04-workflows/16-frame.md).
- **Verify stage** — downstream stage after draft. Verifier runs `cite-check` to trace every substantive claim back to a claim note; failed traces spawn `gap:` cards in the upstream queue, closing the gap-loop into discovery. Output is a `[!verification]` callout at the top of the draft. See [04-workflows/17-verify.md](../04-workflows/17-verify.md).
- **Revise stage** — downstream stage between verify and export. Operator addresses verification findings — soften the claim, pursue the gap, or accept-soft — until the verify→revise loop closes. Export is blocked on revise's closure. See [04-workflows/18-revise.md](../04-workflows/18-revise.md).

## Note types

- **Source note** (`source-note`) — one note per citable source (paper, dataset, report, software), lives in `20-sources/01-literature/`.
- **Claim note** (`claim-note`) — a single durable idea, in your own words; lives in `30-synthesis/01-permanent/`.
- **Reference note** (`reference-note`) — a stable reference page promoted from the claim layer; lives in `30-synthesis/02-wiki/`.
- **MOC** — Map of Content, a navigational hub; lives in `30-synthesis/03-moc/`.
- **Fleeting note** — raw capture awaiting promotion or discard; lives in `10-inbox/01-fleeting/`.
- **Synthesis note** — agent-drafted answer to a research question; lives in `10-inbox/02-synthesis/`.
- **Deliverable** — finished export, manuscript, or presentation; lives in `50-deliverables/`.

## Future-direction terms

- **Propagation debts** — queue of dependents that need re-evaluation when a high-traffic note changes (a `claim-note` promoted to `evergreen`, a `reference-note` updated). Materialized by the linter into a dashboard, worked down by the operator.
- **Execution-trace reflection** — retry pattern where the Kanban dispatcher (or a dedicated reflection skill) reads a failure trace and synthesizes a *modified* task packet for the next attempt, rather than re-dispatching the same packet. Strictly per-card scoped; never rewrites prompts or skills.
